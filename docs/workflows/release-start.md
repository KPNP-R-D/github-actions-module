# Release · Start (Reusable)

선택된 서비스들의 커밋 로그를 수집해 Draft를 만들고, Gemini로 릴리즈 노트를 정제한 뒤, Notion 페이지 생성 + manifest 커밋/푸시까지 처리한다.

---

## Contract

### Required permissions
Caller에서:
- `permissions: contents: write`

### Required checkout
- `actions/checkout@v4` with `fetch-depth: 0`
  - 태그/히스토리 기반 previousTag 후보 계산 + 범위 log 추출 때문에 필요

---

## workflow_call inputs

### Required
- `catalog_path` (string)  
  서비스 카탈로그 JSON 경로  
  e.g. `.github/release/service-catalog.json`
- `prompts_dir` (string)  
  프롬프트 루트 디렉토리  
  e.g. `prompts`
- `inputs_json` (string)  
  caller의 `toJson(inputs)` 결과(서비스 체크박스 읽는 용도)
- `notion_category` (string)  
  Notion DB 속성 `구분`의 select 값 (예: `백엔드`)

### Optional
- `manifest_dir` (string, default: `releases`)
- `previous_tags` (string, default: `""`)
- `release_key` (string, default: `""`)
- `note` (string, default: `""`)

---

## workflow_call secrets

Required:
- `GEMINI_API_KEY`
- `NOTION_TOKEN`
- `NOTION_DB_ID`

Optional:
- `NOTION_DS_NAME`

자세한 의미/준비는: [Shared secrets/permissions](../shared/secrets-permissions.md)

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
````

* `inputKey`: caller workflow_dispatch 체크박스 key
* `tagPrefix`: 태그 prefix. 태그 목록은 `"<tagPrefix>/v*"` 패턴으로 조회한다.
* `servicePaths`: 서비스별 커밋 로그 대상 경로
* `sharedPaths`: 공통 모듈 커밋 로그 대상 경로 (선택 서비스들의 sharedPaths를 merge)

---

## previous_tags override

형식: CSV `key=value`
key는 `서비스명` 또는 `inputKey` 둘 다 허용.

예시:

* `Admin=kpnp-admin/v1.2.3,Champs=kpnp-champs/v2.0.1`
* `admin=kpnp-admin/v1.2.3,champs=kpnp-champs/v2.0.1`
* 커밋 SHA 강제:

  * `admin=1a2b3c4d...`

규칙:

* value가 `<tagPrefix>/...`로 시작하면 tag 존재 검증 후 사용
* 그 외는 commit으로 취급하고 commit 존재 검증 후 사용
* 선택하지 않은 서비스에 대한 override가 들어오면 실패

---

## What it does

### Outputs / Side effects

* `${manifest_dir}/${releaseKey}.yml` 생성 (기본: `releases/<YYYY-MM-DD-XXX>.yml`)
* GitHub Step Summary에:

  * included services / shared paths / previousTag resolution
  * Shared+Services git log 커맨드/결과 디버그
  * Gemini 결과
* Notion 페이지 생성 + 본문 블록 append + Raw Draft(toggle) append
* git commit + push:

  * `chore(release): init draft [RN-BOT][RN-START] (${releaseKey})`

---

## Steps detail

### 0) Checkout

* `fetch-depth: 0`로 태그/히스토리 전체 확보

### 1) Install python deps

* `pyyaml` 설치 (manifest YAML read/write)

### 2) Resolve releaseKey

* `inputs.release_key`가 있으면 그대로 사용
* 없으면 `YYYY-MM-DD-<run_number 3자리>`로 생성

### 3) Create manifest (paths + shared + previousTag candidates)

* `inputs_json`에서 선택된 서비스(inputKey=true) 목록을 만든다.
* `catalog_path`의 서비스 설정을 읽는다.
* 서비스별 `previousTagCandidates`를 생성한다:

  * `git tag --list "<tagPrefix>/v*" --sort=-version:refname | head -n 20`
* `previous_tags`가 있으면 서비스별 `previousTag`를 강제 지정한다:

  * value가 `<tagPrefix>/...`면 tag 존재 검증
  * 아니면 commit 존재 검증 후 `git rev-parse <rev>^{commit}` 정규화
* 선택된 서비스들의 `sharedPaths`를 merge 해서 `shared.paths`에 기록한다.
* manifest 주요 필드:

  * `releaseKey`, `status=DRAFT`, `services`, `shared.paths`, `shared.policy.mode=separate_section_only`
  * `meta.baselineHeadSha=HEAD`, `meta.createdAt(UTC)`, `meta.note`, `meta.includedServices`
* `${manifest_dir}/${releaseKey}.yml`로 저장
* Step Summary에 included/shared/previousTag 해석 결과를 출력

### 4) Build draft (commit logs)

* `git fetch --tags --force`
* manifest를 읽어 `baselineHeadSha`, `includedServices`, 서비스별 `previousTag/paths`, `shared.paths`를 가져온다.
* `global_prev` 선정:

  1. included 서비스들의 previousTag 중 가장 오래된(tag timestamp 기준)
  2. 없으면 서비스별 `paths`에서 `git log -n 80`으로 찾은 가장 오래된 커밋 SHA
* Shared logs 범위:

  * global_prev 있으면 `global_prev..baseline`
  * 없으면 `baseline` 단일
* Service logs 범위:

  * previousTag 있으면 `previousTag..baseline`
  * 없으면 `baseline` 단일
* 커밋 메시지 파싱:

  * subject + body를 bullet 형태로 변환
  * `[RN-BOT]` 포함 커밋은 제외
* 결과 텍스트를 `steps.build_draft.outputs.draft`로 export
* Step Summary에 디버그(커맨드/출력)를 toggle로 append

### 5) Build Gemini content

* `${prompts_dir}`에서 다음 파일을 읽는다:

  * `gemini/release-notes.system.md` (system prompt)
  * `gemini/release-notes.input.md` (입력 포맷)
  * `release-notes.template.md` (Notion 템플릿)
* `release-notes.input.md`에 `{{TEMPLATE}}`, `{{COMMITS}}`를 치환해 Gemini 입력 `content`를 만든다.
* `prompt`, `content`를 step output으로 export

### 6) Gemini refine

* `actions/gemini-generate` 호출:

  * `system_instruction = steps.prompt.outputs.prompt`
  * `content = steps.prompt.outputs.content`
* 출력 `steps.llm.outputs.text`를 생성

### 7) Append Gemini output to Step Summary

* Step Summary에 Gemini 결과를 그대로 기록

### 8) Create Notion page (draft)

* DB에서 `data_source_id`를 resolve한다:

  * `NOTION_DS_NAME` 있으면 해당 name 매칭, 없으면 첫 번째 data_source 사용
* “배포 버전” rich_text를 만든다:

  * 각 서비스의 `previousTag` 기준으로 다음 태그를 계산(없으면 `-`)
* “프로젝트” multi_select를 만든다:

  * includedServices를 그대로 사용
* Notion page 생성 properties:

  * 제목=releaseKey, 구분=inputs.notion_category, 배포 날짜=today, 배포 버전=rich_text, 상태=배포예정, 프로젝트=multi_select
* 본문 블록 append:

  * 데이터2(=Gemini 결과)를 상단에 append (hr `---` 제거)
  * divider 1개
  * toggle(`Raw Draft (Data 1)`) 생성
  * toggle children에 데이터1(=draft) append
  * 95개 chunk로 쪼개서 PATCH (Notion blocks limit 회피)
* manifest의 `meta.notionPageId`에 pageId 저장

### 9) Commit

* `${manifest_dir}/${releaseKey}.yml`을 커밋/푸시
* 커밋 메시지:

  * `chore(release): init draft [RN-BOT][RN-START] (${releaseKey})`

---

## Caller example

```yaml
jobs:
  call-release-start:
    uses: 5n-project/github-actions-module/.github/workflows/release-start.reusable.yml@v1.0.0
    with:
      catalog_path: .github/release/service-catalog.json
      prompts_dir: prompts
      manifest_dir: releases

      inputs_json: ${{ toJson(inputs) }}
      previous_tags: ${{ inputs.previous_tags }}
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

### `contents: write` 권한 관련 에러

* caller에 `permissions: contents: write` 필요
* org 정책에서 reusable 호출 권한 제한될 수 있음

### tag/commit 조회 실패

* `fetch-depth: 0` 확인
* 워크플로 내부에서 `git fetch --tags --force` 실행됨

### Notion data_source_id resolve 실패

* `NOTION_DB_ID`가 올바른 DB인지 확인
* `NOTION_DS_NAME`을 쓰는 경우 정확히 일치해야 함
