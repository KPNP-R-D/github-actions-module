# Release · Finalize (Reusable)

릴리즈를 최종 확정한다.

동작:
1) manifest의 모든 서비스 결과가 `DEPLOYED|SKIPPED`인지 Gate로 검증  
2) manifest를 `FINALIZED`로 변경하고 `bundle/<releaseKey>` 태그 정보를 기록  
3) manifest를 커밋/푸시  
4) `bundle/<releaseKey>` annotated tag 생성/푸시 (멱등 + 검증)  
5) Notion에서 해당 페이지를 찾아 `상태=배포완료`로 변경  
6) GitHub Release를 tag 기준으로 생성(Generate notes)

---

## Contract

### Required permissions
reusable workflow 권한(선언됨):
- `contents: write`
- `pull-requests: read` (현재 YAML 내에서 직접 사용하지 않지만 선언됨)

Caller도 동일 권장:
```yaml
permissions:
  contents: write
  pull-requests: read
````

### Required checkout

* `actions/checkout@v4` + `fetch-depth: 0`

---

## workflow_call inputs

### `release_key` (string, required)

Finalize 대상 릴리즈 키. 파일 경로는 `${manifest_dir}/${release_key}.yml`로 고정된다.

### `manifest_dir` (string, optional, default: `releases`)

manifest 디렉토리.

### `notion_gubun_name` (string, required)

Notion에서 페이지를 찾기 위한 `구분` 값.

* 예: `백엔드`

---

## workflow_call secrets

### `notion_token` (required)

Notion Integration Token

### `notion_db_id` (required)

Notion Database ID

---

## Steps detail

### 0) Resolve metadata path

* `${manifest_dir}/${release_key}.yml` 파일이 없으면 실패

### 1) Gate - validate service statuses

manifest의 모든 서비스에 대해:

* `services.<svc>.result`가 `DEPLOYED` 또는 `SKIPPED`여야 함
* 하나라도 아니면 실패 (FAILED/None 등)

### 2) Update metadata status to FINALIZED

manifest 수정:

* `status = FINALIZED`
* `bundle.tag = "bundle/<release_key>"`
* `meta.finalizedAt = <UTC ISO8601 Z>`

### 3) Commit & push (idempotent)

* 변경이 없으면 커밋 생략
* 커밋 메시지:

  * `chore(release): finalize [RN-BOT][RN-FINALIZE] (<release_key>)`
* 커밋 후 `sha` output으로 HEAD SHA를 기록

### 4) Create & push bundle tag (idempotent + verify)

* TAG: `bundle/<release_key>`
* 이미 태그가 존재하면:

  * 태그가 현재 HEAD SHA와 같아야 통과
  * 다르면 실패(태그가 다른 커밋을 가리키는 상황)
* 없으면 annotated tag 생성 후 push

### 5) Notion 상태 업데이트 (배포완료)

Notion DB에서 다음 조건의 페이지를 찾는다:

* 제목(title property) == `<release_key>`
* `구분`(select 또는 multi_select) == `<notion_gubun_name>`

찾은 페이지에 대해:

* `상태`(select) 를 `배포완료`로 설정
* 이미 `배포완료`면 멱등 처리로 종료

구현 디테일:

* Notion API 버전: `2025-09-03`
* `구분` property는 `select` 또는 `multi_select`만 지원(그 외 타입이면 skip)
* DB가 여러 `data_sources`를 가지면, 모든 data_source를 순회하며 페이지를 검색

### 6) GitHub Release 생성 (generate notes)

* tag: `bundle/<release_key>`
* 이미 release가 있으면 멱등 처리로 종료
* 없으면:

  * `gh release create <tag> --title "Release <release_key>" --generate-notes`

---

## Caller example

```yaml
jobs:
  call-release-finalize:
    uses: 5n-project/github-actions-module/.github/workflows/release-finalize.reusable.yml@v1.0.0
    with:
      release_key: ${{ inputs.release_key }}
      manifest_dir: releases
      notion_gubun_name: "백엔드"
    secrets:
      notion_token: ${{ secrets.KPNP_NOTION_TOKEN }}
      notion_db_id: ${{ secrets.KPNP_NOTION_DB_ID }}
```

Caller workflow:

```yaml
permissions:
  contents: write
  pull-requests: read
```

---

## Troubleshooting

### `metadata not found`

* `${manifest_dir}/${release_key}.yml` 파일이 실제로 존재하는지 확인
* Start 단계에서 key/dir이 맞는지 확인

### `not ready to finalize. invalid service results`

* Report 단계에서 모든 서비스가 `DEPLOYED` 또는 `SKIPPED`가 되도록 정리해야 finalize 가능

### `tag already exists but points to different commit`

* 동일한 `bundle/<release_key>` 태그가 이미 존재하고 다른 커밋을 가리킴
* 현재 로직은 “태그 재지정(force)”을 금지하므로 실패가 정상

  * 해결: release_key 충돌 방지 / 기존 태그 정리(운영 정책 필요)

### Notion page not found

* Notion DB에서:

  * 제목이 정확히 release_key인지
  * `구분` 값이 notion_gubun_name과 일치하는지
  * `구분` 속성 타입이 select/multi_select인지 확인

### GitHub Release 생성 실패

* `GH_TOKEN` 권한/레포 접근 권한 확인
* tag가 원격에 존재하는지 확인
