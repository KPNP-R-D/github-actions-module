# Release · Generate Notes & Tag (Reusable)

선택된 서비스들의 커밋 로그를 수집해 Draft를 만들고, Gemini로 릴리즈 노트를 정제한 뒤, Notion 페이지 생성 + `bundle/<releaseKey>` 태그 생성/푸시 + GitHub Release 생성까지 처리한다.

---

## Contract

### Required permissions

Caller에서:

* `permissions: contents: write`

  * 태그 생성/푸시, release 생성에 사용

### Required checkout

* `actions/checkout@v4` with `fetch-depth: 0`

  * 태그/히스토리 기반 previousTag 계산 + 범위 log 추출 때문에 필요

---

## workflow_call inputs

### Required

* `catalog_path` (string)
  서비스 카탈로그 JSON 경로
  e.g. `.github/release/service-catalog.json`
* `prompts_dir` (string)
  프롬프트 루트 디렉토리
  e.g. `prompts`
* `inputs_json` (string)
  caller의 `toJson(inputs)` 결과(서비스 체크박스 읽는 용도)
* `notion_category` (string)
  Notion DB 속성 `구분`의 select 값 (예: `백엔드`)

### Optional

* `manifest_dir` (string, default: `releases`)

  * 현재 워크플로에서는 **YML 파일을 생성하지 않음**. (입력 정의는 유지)
* `current_tags` (string, default: `""`)
  선택된 서비스들의 현재 배포 버전 매핑. 사실상 필수값.
  형식: `key=value,key=value` (CSV)
* `release_key` (string, default: `""`)
* `note` (string, default: `""`)

---

## workflow_call secrets

Required:

* `GEMINI_API_KEY`
* `NOTION_TOKEN`
* `NOTION_DB_ID`

Optional:

* `NOTION_DS_NAME`

---

## Catalog JSON spec

`catalog_path`는 아래 형태(JSON object)를 기대한다.

```json
{
  "Admin": {
    "inputKey": "admin",
    "tagPrefix": "kpnp-admin",
    "servicePaths": ["service/kpnp-admin"],
    "sharedPaths": ["service/core"]
  },
  "Champs": {
    "inputKey": "champs",
    "tagPrefix": "kpnp-champs",
    "servicePaths": ["service/kpnp-champs"],
    "sharedPaths": ["service/core", "service/security"]
  }
}
```

* `inputKey`: caller workflow_dispatch 체크박스 key
* `tagPrefix`: 태그 prefix. 태그 목록은 `"<tagPrefix>/v*"` 패턴으로 조회
* `servicePaths`: 서비스별 커밋 로그 대상 경로
* `sharedPaths`: 공통 모듈 커밋 로그 대상 경로(선택 서비스들의 sharedPaths merge)

---

## current_tags mapping (핵심 입력)

형식: CSV `key=value`

* key는 `서비스명` 또는 `inputKey` 둘 다 허용.
* value는 **태그** 또는 **git revision(커밋 SHA/브랜치/HEAD 등)** 허용.
* 검증:

  * value가 tag로 존재하면 tag 모드
  * 아니면 commit으로 해석 가능한 revision이면 commit 모드
  * 둘 다 아니면 실패
* 선택하지 않은 서비스가 포함되면 실패
* 선택된 서비스가 누락되면 실패

예시:

* 태그로 지정

  * `Admin=kpnp-admin/v1.2.3,Champs=kpnp-champs/v2.0.1`
  * `admin=kpnp-admin/v1.2.3,champs=kpnp-champs/v2.0.1`
* 커밋/리비전으로 지정

  * `admin=1a2b3c4d...`
  * `champs=origin/main`

---

## What it does

### Outputs / Side effects

* Step Summary에:

  * included services / shared paths / baseline HEAD / globalPrevSha / 서비스별 previousTag
  * Shared+Services git log 커맨드/결과 디버그
  * Gemini 결과
* Notion 페이지 생성 + 본문 블록 append + Raw Draft(toggle) append

  * `page_id`를 step output으로 제공 (`steps.notion_draft.outputs.page_id`)
* Git tag 생성/푸시:

  * `bundle/<releaseKey>` annotated tag (idempotent + commit mismatch 검증)
* GitHub Release 생성:

  * tag=`bundle/<releaseKey>`로 release 생성 (`gh release create --generate-notes`)
  * 이미 존재하면 스킵

---

<details>
<summary><strong>Steps detail</strong></summary>

### 0) Checkout

* `fetch-depth: 0`로 태그/히스토리 전체 확보

### 1) Install python deps

* `pyyaml` 설치 (현재 워크플로는 manifest를 JSON으로 다루지만 공용 의존성 유지)

### 2) Resolve releaseKey

* `inputs.release_key`가 있으면 그대로 사용
* 없으면 `YYYY-MM-DD-<run_number 3자리>`로 생성

### 3) Create manifest (compute config only; no yml)

YML 파일을 만들지 않고 **manifest JSON**을 계산해 step output으로 export.

* `inputs_json`에서 선택 서비스(inputKey=true) 목록 생성
* `current_tags` 파싱 + 서비스별 current(tag/commit) 확정
* 서비스별 `previousTag` 계산

  * current가 tag면: `tagPrefix/v*` 리스트에서 현재 tag의 **바로 이전 tag**
  * current가 commit이면: `previousTag` 없음(= fallback 대상)
* 선택 서비스들의 `sharedPaths` merge → `shared.paths`
* `globalPrevSha` 선정(Shared 범위용)

  * 서비스별 후보를 모아 **timestamp가 가장 오래된 sha** 선택
  * 후보 규칙:

    1. `previousTag` 있으면 그 tag의 sha
    2. 없으면 `servicePaths`에서 `git log -n 80`의 “가장 오래된 커밋 sha”
* Step Summary에 included/shared/baseline/global prev/previous tags 출력
* 결과: `steps.manifest.outputs.manifest_json`

### 4) Build draft (commit logs)

* `git fetch --tags --force`
* manifest JSON에서 `baselineHeadSha/globalPrevSha/shared.paths/services` 읽음
* Shared logs 범위:

  * globalPrevSha 있으면 `globalPrevSha..baseline`
  * 없으면 `baseline`
* Service logs 범위:

  * currentType=tag:

    * previousTag 있으면 `previousTag..currentTag`
    * 없으면 `git log -n 80 currentTag`
  * currentType=commit:

    * `git log -n 80 currentRef` (정규화된 full sha)
* 커밋 메시지 파싱:

  * subject + body를 bullet로 변환
  * body는 indent 기반으로 1~3 depth 유지
  * `[RN-BOT]` 포함 커밋 제외
* output:

  * `draft`, `has_commits`, `commit_count`, `service_commit_counts`
* Step Summary에 debug(cmd/output + draft)를 `<details>`로 append

### 5) Fail job if no commit messages

* `has_commits != true`면 `exit 1` (빈 릴리즈 차단)

### 6) Build Gemini content

* `${prompts_dir}`에서 다음 파일 로드:

  * `gemini/release-notes.system.md`
  * `gemini/release-notes.input.md`
  * `release-notes.template.md`
* input 템플릿에 `{{TEMPLATE}}`, `{{COMMITS}}` 치환 → Gemini `content`
* 결과: `steps.prompt.outputs.prompt/content`

### 7) Gemini refine

* `actions/gemini-generate` 호출

  * `system_instruction = prompt`
  * `content = content`
* 결과: `steps.llm.outputs.text`

### 8) Append Gemini output to Step Summary

* Step Summary에 Gemini 결과 그대로 기록

### 9) Create Notion page (draft)

* DB에서 `data_source_id` resolve

  * `NOTION_DS_NAME` 있으면 name 매칭, 없으면 첫 data_source
* “배포 버전” rich_text

  * 서비스별 `current` 표시
  * commit이면 full sha를 7자리로 축약해 code 스타일로 표시
* “프로젝트” multi_select

  * includedServices 그대로
* Notion page 생성 properties:

  * 제목=releaseKey, 구분=notion_category, 배포 날짜=today, 배포 버전, 프로젝트
* 본문 append

  * 데이터2(=Gemini) 상단 append (hr `---` 제거)
  * divider 1개
  * toggle `Raw Draft (Data 1)` 생성
  * toggle children에 데이터1(=draft) append
  * 95개 chunk로 PATCH (block 제한 회피)
* output: `steps.notion_draft.outputs.page_id`

### 10) Create & push bundle tag (idempotent + verify)

* tag: `bundle/<releaseKey>`
* 존재하면:

  * tag가 가리키는 commit != HEAD → 실패(충돌 방지)
  * 같으면 notice 후 스킵
* 없으면:

  * annotated tag 생성 + push

### 11) Create GitHub Release (generate notes)

* `gh release view <tag>`로 존재 여부 확인
* 없으면 `gh release create <tag> --title ... --generate-notes`
* 있으면 notice 후 종료

</details>

---

## Caller example

```yaml
jobs:
  call-release-generate-notes-and-tag:
    uses: 5n-project/github-actions-module/.github/workflows/release-generate-notes-and-tag.reusable.yml@v1.0.0
    with:
      catalog_path: .github/release/service-catalog.json
      prompts_dir: prompts
      manifest_dir: releases

      inputs_json: ${{ toJson(inputs) }}
      current_tags: ${{ inputs.current_tags }}

      release_key: ${{ inputs.release_key }}
      note: ${{ inputs.note }}

      notion_category: "백엔드"
    secrets:
      GEMINI_API_KEY: ${{ secrets.GEMINI_API_KEY }}
      NOTION_TOKEN: ${{ secrets.KPNP_NOTION_TOKEN }}
      NOTION_DB_ID: ${{ secrets.KPNP_NOTION_DB_ID }}
      NOTION_DS_NAME: ${{ secrets.KPNP_NOTION_DS_NAME }}
```

Caller workflow에는 아래도 같이 있어야 함:

```yaml
permissions:
  contents: write
```

---

## Troubleshooting

### `current_tags` 매핑 에러

* `current_tags`가 빈 값이면 실패.
* 선택 서비스 누락/선택되지 않은 서비스 포함/중복 key면 실패.
* key는 `서비스명` 또는 `inputKey`만 허용.

### tag/commit 조회 실패

* `fetch-depth: 0` 확인
* `current_tags`의 value가 실제 tag 또는 commit rev로 해석 가능해야 함

### Notion data_source_id resolve 실패

* `NOTION_DB_ID`가 올바른 DB인지 확인
* `NOTION_DS_NAME` 사용 시 정확히 일치해야 함

### 빈 릴리즈로 실패(`Fail job if no commit messages`)

* 범위 내 커밋이 없거나 `[RN-BOT]` 필터로 전부 제외된 케이스.
* Step Summary의 debug(각 서비스 git log cmd/출력)로 범위 확인.
