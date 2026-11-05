# Reusable Notify (Telegram) — v2

PR·배포 알림을 **전 레포 공통 서식**으로 텔레그램에 보낸다. 호출 레포는 "언제 보낼지"만 정하고, 메시지 내용은 이 모듈이 GitHub 이벤트와 Actions API 에서 채운다.

```yaml
uses: KPNP-R-D/github-actions-module/.github/workflows/notify-telegram.yml@v2
```

> v1(`push | pr | merge | deploy`, 평문)은 `@v1` 태그에 그대로 남아 있다. v2 는 입력·Secret·서식이 다르므로 레포별로 옮긴다.

---

## 메시지 서식

- 이모지는 🧩(PR) ✅(배포 성공) ❌(배포 실패) 세 개만 쓴다.
- 첫 줄은 굵은 머리말, 이후 각 줄은 `항목: 값` 이다. 값이 없는 항목은 줄째 빠진다.
- `parse_mode=HTML` 로 보내고 모든 값은 HTML 이스케이프한다. 링크 미리보기는 끈다.

### PR

```
🧩 PR 열림                       ← PR 재오픈 / PR 리뷰 준비 완료
레포: KPNP-Champs-Front
제목: #123 [fix] 품새 - …         (PR 링크)
브랜치: fix/referee → dev
작성자: jw
리뷰어: mandykr, front           (이벤트 시점의 requested_reviewers · requested_teams)
변경: 커밋 3 · 파일 5 · +120 −30
라벨: bug, frontend
```

### 배포

```
✅ 배포 성공                      ← ❌ 배포 실패
레포: KPNP-Champs-Front
프로젝트: champs-front            (inputs.project)
환경: dev                         (inputs.environment)
브랜치: dev
커밋: 9ab47c8 [docs] CI - … 외 2건 (커밋 링크 · push 에 묶인 커밋 수 · 20건부터 「이상」)
메시지: 긴급 패치                  (inputs.message 첫 줄)
태그: kpnp-champs/v1.2.3          (inputs.tag)
실패 단계: deploy › 기동 확인       (실패일 때 · 처음 실패한 job › step)
실행자: mandykr (수동 실행 · 재실행 2회차)
소요 시간: 4분 42초               (실행 시작부터 알림까지)
실행 로그: deploy-champs-front-dev #57 (실행 링크)
서비스: https://dev.example.com    (inputs.url)
```

---

## 보내는 조건

| event_kind | 호출 이벤트 | 보냄 | 보내지 않음 (job 은 성공) |
|---|---|---|---|
| `pr` | `pull_request` · `pull_request_target` | `opened` · `reopened` · `ready_for_review` | Draft PR, 그 밖의 action(`synchronize` 등), `pull_request` 이벤트의 Dependabot·fork PR (Secret 이 넘어오지 않는다) |
| `deploy` | 아무 이벤트 | `result` = `success` · `failure` | `result` = `cancelled` · `skipped` |

실패로 끝나는 경우 — 잘못 호출했거나 설정이 빠졌다는 뜻이다.

- `event_kind` 가 `pr` · `deploy` 가 아니다
- `event_kind: pr` 를 pull_request 가 아닌 이벤트에서 호출했다
- `deploy` 의 `result` 가 비었거나 위 네 값이 아니다
- `TELEGRAM_TOKEN` 또는 대상 채팅방 Secret 이 비어 있다 — **다른 방으로 대신 보내지 않는다**
- 텔레그램 API 가 오류를 돌려줬다 (응답 본문을 로그에 남긴다)

---

## 입력

| Input | 대상 | 설명 |
|---|---|---|
| `event_kind` | 공통 · 필수 | `pr` \| `deploy` |
| `result` | deploy · 필수 | `success` \| `failure` \| `cancelled` \| `skipped` |
| `project` | deploy | 프로젝트/서비스명 |
| `environment` | deploy | `dev` · `stg` · `prod` 등 |
| `message` | deploy | 배포 메모 (첫 줄만 쓴다) |
| `tag` | deploy | 배포한 태그 |
| `url` | deploy | 배포된 서비스 주소 |

## Secret

`secrets: inherit` 로 넘긴다.

| Secret | 쓰는 곳 | 범위 |
|---|---|---|
| `TELEGRAM_TOKEN` | 공통 | 조직 Secret (KPNP_Notifier_bot) |
| `TELEGRAM_TO_PR` | pr | **레포 Secret** — PR 방은 레포마다 다르다 |
| `TELEGRAM_TO_DEPLOY` | deploy | 조직 Secret (「배포」 방) |

봇이 대상 채팅방에 들어가 있어야 한다. 조직 Secret 은 Repository access 에 호출 레포가 포함돼야 한다.

⚠️ `TELEGRAM_TO_PR` 을 조직 Secret 으로 만들지 않는다 — `secrets: inherit` 는 레포 Secret 이 없으면 같은 이름의 조직 Secret 을 넘기므로, 레포 Secret 을 빠뜨린 레포의 PR 이 그 방으로 간다.

## 권한

호출 job 에 `permissions` 를 적으면 아래만큼 준다. 적지 않으면 호출 워크플로의 권한(최상위 `permissions`, 없으면 레포 기본값)을 따른다.

| 권한 | 없으면 |
|---|---|
| `actions: read` | 배포 알림에서 「소요 시간」·「실패 단계」 줄이 빠진다 (경고만 남기고 전송은 한다) |
| `contents: read` | push 가 아닌 배포(수동 실행 등)에서 커밋 제목이 빠진다 |

---

## 호출 예시

### PR 알림 — `.github/workflows/notify-pr.yml`

```yaml
name: Notify PR

on:
  pull_request:
    types: [opened, reopened, ready_for_review]

jobs:
  notify:
    uses: KPNP-R-D/github-actions-module/.github/workflows/notify-telegram.yml@v2
    with:
      event_kind: pr
    secrets: inherit
```

### 배포 알림 — 배포 워크플로우의 마지막 job

```yaml
  notify:
    needs: [build, deploy]
    if: always()
    permissions:
      contents: read
      actions: read
    uses: KPNP-R-D/github-actions-module/.github/workflows/notify-telegram.yml@v2
    with:
      event_kind: deploy
      project: champs-front
      environment: dev
      # 앞 job 이 실패하면 deploy 는 skipped 가 된다 — 실패를 failure 로 접어서 넘긴다
      # needs 에 상류 job 을 전부 적는다. 빠진 job 의 실패는 skipped 로 묻혀 알림이 가지 않는다
      result: ${{ contains(needs.*.result, 'failure') && 'failure' || needs.deploy.result }}
    secrets: inherit
```

---

## v1 에서 옮기기

| v1 | v2 |
|---|---|
| `event_kind: push` · `merge` | 없다 — 지우거나 실제 배포 job 의 `deploy` 알림으로 바꾼다 |
| `event_kind: deploy` + `result: success` 고정값 | 실제 결과를 넘긴다 (위 예시의 `result` 식) |
| PR 알림을 `synchronize`·`closed` 에도 호출 | `types: [opened, reopened, ready_for_review]` 만 — 그 밖의 action 은 모듈이 건너뛴다 |
| `TELEGRAM_TO` 로 대신 보냄 | 대신 보내지 않는다 — `TELEGRAM_TO_PR`(레포) · `TELEGRAM_TO_DEPLOY`(조직) 를 등록한다 |
| Secret 을 하나씩 전달 | `secrets: inherit` |
| `permissions: contents: read` | 배포 알림 job 에 `actions: read` 를 더한다 |
