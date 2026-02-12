# Reusable Notify (Telegram Router)

`event_kind`(push/pr/merge/deploy)에 따라 **텔레그램 메시지 포맷을 통일**하고, 이벤트별 채널로 **라우팅해서 전송**한다. 메시지 본문은 가능한 데이터 소스(입력 message → GitHub payload → git log)를 우선순위로 선택해 최대한 빈 메시지를 피한다.

---

## Contract

### Required permissions

* 기본적으로 읽기만 수행하지만, 이 워크플로는 caller repo를 checkout해서 `git log`를 읽는다.
* 권장:

  * `permissions: contents: read`

> 별도 OIDC/AWS 권한 없음.

### Required checkout

* 내부에서 `actions/checkout@v4` 수행

  * `repository: ${{ github.repository }}`
  * `ref: ${{ github.sha }}`
  * `fetch-depth: 0` (git log 안정성)

### Prerequisites

* Telegram Bot Token 필요
* 수신 채널(chat_id) 라우팅을 위한 secrets 구성 필요
* 실제 전송은 composite action `actions/notify-telegram@v1`에 위임됨

---

## workflow_call inputs

### Required

| Input        | Type   | Description                     |
| ------------ | ------ | ------------------------------- |
| `event_kind` | string | `push \| pr \| merge \| deploy` |

### Optional (deploy payload)

| Input         | Type   | Description           |
| ------------- | ------ | --------------------- |
| `project`     | string | deploy 전송용 프로젝트/서비스명  |
| `environment` | string | dev/stg/prod 등        |
| `result`      | string | `success` / `failure` |
| `message`     | string | deploy 메시지(우선순위 1)    |
| `tag`         | string | 배포 태그(선택)             |

> 주의: 스크립트에서 `inputs.image`를 참조하지만 **inputs에 정의되어 있지 않음**. 현재 상태 그대로면 `${{ inputs.image }}`는 빈 값으로 평가될 가능성이 높다(또는 lint/validation에 걸릴 수 있음). 아래 “Troubleshooting” 참고.

---

## workflow_call secrets

Required:

| Secret           | Description           |
| ---------------- | --------------------- |
| `TELEGRAM_TOKEN` | Telegram bot token    |
| `TELEGRAM_TO`    | 기본 chat_id (fallback) |

Optional (event별 라우팅. 없으면 `TELEGRAM_TO`로 fallback):

| Secret               | Used when `event_kind` = | Description       |
| -------------------- | -----------------------: | ----------------- |
| `TELEGRAM_TO_PUSH`   |                     push | push 전용 chat_id   |
| `TELEGRAM_TO_PR`     |                       pr | PR 전용 chat_id     |
| `TELEGRAM_TO_MERGE`  |                    merge | merge 전용 chat_id  |
| `TELEGRAM_TO_DEPLOY` |                   deploy | deploy 전용 chat_id |

---

## What it does

### Outputs / Side effects

* 이벤트 종류에 맞는 메시지 포맷 생성 후 텔레그램 전송
* 라우팅 규칙:

  * `event_kind`별 chat_id secret 우선
  * 없으면 `TELEGRAM_TO` 사용

---

## Message spec

### 공통 메타

* `Repo`: `${{ github.repository }}`
* `By`: `${{ github.actor }}`
* `Ref`: `${{ github.ref_name }}`
* `Workflow Run`: `https://github.com/<repo>/actions/runs/<run_id>`

### push 메시지

* 제목: `git log -1 --pretty=%s` (fallback 포함)
* 링크:

  * Recent Commits: `https://github.com/<repo>/commits/<ref>`

### pr 메시지

* PR 번호/제목/작성자/URL
* from(head) → base

### merge 메시지

* PR 번호/제목/merger/URL
* from(head) → base

### deploy 메시지

* 결과에 따라 아이콘/상태:

  * success → ✅ SUCCESS
  * else → ❌ FAILED
* message 우선순위(최종 1줄로 정리):

  1. `inputs.message`
  2. `github.event.head_commit.message`
  3. `github.event.pull_request.title`
  4. `git log -1 --pretty=%s`
  5. `"deploy by <actor>"`
* 추가 필드(있을 때만 append):

  * `Tag: <inputs.tag>`
  * `Image: <inputs.image>` (현재 inputs 미정의)
* 마지막에 Workflow Run 링크

---

<details>
<summary><strong>Steps detail</strong></summary>

### 0) Checkout caller repo

* `actions/checkout@v4`로 caller repo의 해당 sha checkout
* `fetch-depth: 0`

### 1) Build unified message

* `event_kind`에 따라 case 분기해서 텍스트 생성
* 결과를 multi-line output(`text<<EOF`) 형태로 `GITHUB_OUTPUT`에 기록

### 2) Decide chat target by event_kind

* `TELEGRAM_TO_<KIND>`가 있으면 사용
* 없으면 `TELEGRAM_TO` fallback
* 결과를 `steps.route.outputs.to`로 export

### 3) Send Telegram

* `actions/notify-telegram@v1` 호출
* `bot_token`, `chat_id`, `message` 전달

</details>

---

## Caller example

### 1) Deploy 알림

```yaml
jobs:
  notify_deploy:
    permissions:
      contents: read
    uses: 5n-project/github-actions-module/.github/workflows/notify.reusable.yml@v1.0.0
    with:
      event_kind: deploy
      project: kpnp-admin
      environment: prod
      result: success
      message: "ECS rollout completed"
      tag: "kpnp-admin/v1.2.3"
    secrets:
      TELEGRAM_TOKEN: ${{ secrets.TELEGRAM_TOKEN }}
      TELEGRAM_TO: ${{ secrets.TELEGRAM_TO }}
      TELEGRAM_TO_DEPLOY: ${{ secrets.TELEGRAM_TO_DEPLOY }}
```

### 2) PR 알림

```yaml
jobs:
  notify_pr:
    permissions:
      contents: read
    uses: 5n-project/github-actions-module/.github/workflows/notify.reusable.yml@v1.0.0
    with:
      event_kind: pr
    secrets:
      TELEGRAM_TOKEN: ${{ secrets.TELEGRAM_TOKEN }}
      TELEGRAM_TO_PR: ${{ secrets.TELEGRAM_TO_PR }}
      TELEGRAM_TO: ${{ secrets.TELEGRAM_TO }}
```

---

## Troubleshooting

### deploy에서 `Image:`가 항상 비어있음 / workflow validation 문제

* 스크립트가 `IMAGE="${{ inputs.image }}"`를 참조하지만, `workflow_call.inputs`에 `image`가 정의되어 있지 않음.
* 조치 옵션:

  1. inputs에 `image` 추가

     ```yaml
     image:
       description: "deployed image ref (deploy 전송용)"
       required: false
       type: string
     ```
  2. 스크립트에서 `inputs.image` 참조 제거(또는 env에서 주입 방식 변경)

### deploy 메시지에 `Image:`가 두 번 출력됨

* 아래 라인이 중복되어 있음:

  ```bash
  [[ -n "${IMAGE:-}" ]] && MSG="${MSG}
  Image:   ${IMAGE}"
  [[ -n "${IMAGE:-}" ]] && MSG="${MSG}
  Image:   ${IMAGE}"
  ```
* 1개만 남겨야 함.

### `event_kind`가 정의된 값인데도 Unsupported로 떨어짐

* 입력 값에 공백/대소문자 혼입 가능.
* caller에서 `event_kind`를 정확히 `push|pr|merge|deploy`로 강제(예: workflow_dispatch choice)하는 게 안전.

### push에서 commit 메시지가 비어있음

* 일부 이벤트/레포 설정에서 `github.event.head_commit.message`가 없을 수 있음.
* 이 워크플로는 `git log -1`로 fallback하므로 checkout이 실패하면 비게 됨.

  * `contents: read` 권한, checkout ref/sha 확인.

### chat_id 라우팅이 기대와 다름

* `TELEGRAM_TO_<KIND>`가 비어있으면 `TELEGRAM_TO`로 fallback.
* secrets 이름 오타(특히 `_MERGE`, `_DEPLOY`) 확인.
