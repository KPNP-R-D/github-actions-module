# Release · Report (Reusable)

선택한 서비스들의 배포 결과(DEPLOYED / SKIPPED / FAILED)를 **manifest에 일괄 반영**하고 커밋/푸시한다.

---

## Contract

### Required permissions
Caller에서:
- `permissions: contents: write`

### Required checkout
- `actions/checkout@v4` + `fetch-depth: 0`

---

## workflow_call inputs

### `manifest_dir` (string, optional, default: `releases`)
manifest 루트 디렉토리. 하위까지 재귀 탐색해서 `.yml/.yaml`을 후보로 수집한다.

### `selected_services` (string, required)
CSV 형태로 서비스 선택.
- 예: `admin,champs`
- 공백 허용: `admin, champs`

주의:
- **manifest의 `services.<svc>.inputKey` 값으로 매칭**한다.
  - 즉 `selected_services`에는 “서비스명(Admin/Champs)”이 아니라 **inputKey(admin/champs)** 를 넣어야 한다.

중복:
- 중복은 제거된다(순서 유지).

### `release_key` (string, optional, default: `""`)
- 비어있으면: 최신 DRAFT manifest 자동 선택
- 있으면: manifest 내부의 `releaseKey`가 동일한 파일을 선택 (파일명 매칭 아님)

### `result` (string, required)
허용값(대문자 기준):
- `DEPLOYED`
- `SKIPPED`
- `FAILED`

그 외면 실패.

---

## Secrets
없음

---

## What it does

### Side effects
- `${manifest_dir}` 하위의 manifest 중 대상 1개를 찾아 업데이트
  - 선택된 서비스들의 `services.<service>.result`를 `RESULT`로 설정
- 변경사항이 있으면 커밋/푸시
  - `chore(release): update release report [RN-BOT][RN-REPORT] (<releaseKey>)`

### Target manifest resolve
- `release_key`가 있으면: **manifest 내부 `releaseKey`가 동일한 파일**을 찾는다
- `release_key`가 비어 있으면: `status=DRAFT`인 manifest 중 **가장 최신 createdAt**(meta.createdAt) 기준 1개를 선택
  - DRAFT가 없으면 실패 (`release_key를 지정해줘`)

### Gate
- 대상 manifest의 `status`가 `DRAFT`가 아니면 실패

---

## Steps detail

### 0) Checkout
- `fetch-depth: 0`로 manifest 파일 접근 + 커밋/푸시 준비

### 1) Setup Python
- Python 3.11 세팅

### 2) Install python deps
- `pyyaml` 설치 (manifest YAML read/write)

### 3) Update manifest (batch)
- `result` 검증:
  - `DEPLOYED|SKIPPED|FAILED` 외 값이면 실패
- `selected_services` 파싱:
  - CSV를 trim
  - 빈 값 제거
  - 중복 제거(순서 유지)
  - 비어있으면 실패
- `${manifest_dir}` 하위 `.yml/.yaml` 재귀 탐색
  - 파일 없으면 실패
- 대상 manifest 선택:
  - `release_key` 있으면 `releaseKey == release_key`인 manifest 선택
  - 없으면 `status == DRAFT` 중 최신 createdAt 선택
  - 대상 manifest의 status가 DRAFT가 아니면 실패
- 서비스 매칭:
  - manifest의 `services`에서 `inputKey -> serviceName` 역인덱스 생성
  - `selected_services`의 각 item이 idx에 없으면 실패
- 결과 반영:
  - 매칭된 서비스들의 `services.<serviceName>.result = RESULT`
- 저장 후 step output으로 `rk=<releaseKey>` 기록

### 4) Commit & push
- git status에 변경이 있을 때만 커밋/푸시
- 커밋 메시지:
  - `chore(release): update release report [RN-BOT][RN-REPORT] (${rk})`
- 변경 없으면 `No changes to commit.` 출력

---

## Caller example

```yaml
jobs:
  call-release-report:
    uses: 5n-project/github-actions-module/.github/workflows/release-report.reusable.yml@v1.0.0
    with:
      manifest_dir: releases
      release_key: ${{ inputs.release_key }}          # optional
      result: ${{ inputs.result }}                    # DEPLOYED|SKIPPED|FAILED
      selected_services: "admin,champs"               # inputKey CSV
````

Caller workflow:

```yaml
permissions:
  contents: write
```

---

## Troubleshooting

### `no DRAFT manifests found`

* `release_key`를 지정해라.
* 또는 Start 단계에서 manifest가 생성/푸시 되었는지 확인.

### `selected services not found in manifest (by inputKey)`

* `selected_services`에 서비스명(Admin 등)을 넣었을 가능성 큼.
* manifest의 `services.<svc>.inputKey` 값(예: admin/champs)을 넣어야 함.

### push 권한 에러

* caller에 `permissions: contents: write` 누락
* org 정책으로 GITHUB_TOKEN write 제한 가능
