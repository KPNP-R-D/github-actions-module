# Notify Telegram (Composite Action)

Telegram Bot API의 `sendMessage`를 호출해서 지정한 채팅방(`chat_id`)에 메시지를 전송한다. 단일 목적(전송 전용) 액션이라 다른 워크플로/재사용 모듈에서 쉽게 래핑해서 쓰는 용도다.

---

## Contract

### Required permissions

* GitHub 권한은 필요 없음 (API 호출은 외부로 `curl` POST만 수행)

### Network

* GitHub-hosted runner에서 `https://api.telegram.org`로 outbound HTTPS 접근 가능해야 함.

### Security notes

* `bot_token`은 반드시 **secrets**로 전달해야 함.
* 메시지 내용에 민감정보(토큰/비번)가 섞이지 않도록 상위 워크플로에서 필터링 필요.

---

## Inputs

### Required

| Input       | Type   | Description                  |
| ----------- | ------ | ---------------------------- |
| `bot_token` | string | Telegram bot token           |
| `chat_id`   | string | Telegram chat id (room/user) |
| `message`   | string | 전송할 메시지 텍스트                  |

Outputs 없음.

---

## What it does

* 아래 요청을 수행:

  * Method: `POST`
  * Endpoint: `https://api.telegram.org/bot<BOT_TOKEN>/sendMessage`
  * Params:

    * `chat_id=<chat_id>`
    * `text=<message>` (URL-encoded)

* 실패 시 `curl` exit code로 step 실패 처리 (단, 현재는 HTTP 4xx/5xx를 exit code로 올리지 않을 수 있음 → 아래 참고)

---

## Implementation detail

```bash
curl -sS -X POST "https://api.telegram.org/bot${{ inputs.bot_token }}/sendMessage" \
  -d chat_id="${{ inputs.chat_id }}" \
  --data-urlencode text="${{ inputs.message }}"
```

* `--data-urlencode`로 텍스트를 URL 인코딩 처리 (개행/특수문자 대응)
* `-sS`로 silent 모드 + 에러 시 메시지 출력

---

## Usage examples

### 1) 단독 호출

```yaml id="g3o7qr"
- name: Send Telegram
  uses: 5n-project/github-actions-module/actions/notify-telegram@v1
  with:
    bot_token: ${{ secrets.TELEGRAM_TOKEN }}
    chat_id: ${{ secrets.TELEGRAM_TO }}
    message: "✅ Build finished: ${{ github.sha }}"
```

### 2) 멀티라인 메시지

```yaml id="0l0j7e"
- name: Send Telegram
  uses: 5n-project/github-actions-module/actions/notify-telegram@v1
  with:
    bot_token: ${{ secrets.TELEGRAM_TOKEN }}
    chat_id: ${{ secrets.TELEGRAM_TO }}
    message: |
      ✅ DEPLOY SUCCESS
      Project: kpnp-admin
      Tag: v1.2.3
      Run: https://github.com/${{ github.repository }}/actions/runs/${{ github.run_id }}
```

---

## Troubleshooting

### 메시지는 전송됐는데 step이 성공/실패가 애매함

* 현재 구현은 `curl`에 `--fail`이 없어서, Telegram API가 HTTP 400/401을 내려도 curl이 0으로 끝날 수 있음.
* 안정적으로 실패 처리하려면 아래 중 하나로 강화 필요:

  1. `curl -f` 또는 `--fail-with-body` 추가
  2. 응답 JSON의 `"ok": true` 여부를 `jq`로 검사

예(강화 버전 개념):

```bash
curl -sS --fail-with-body -X POST ...
```

### `401 Unauthorized`

* `bot_token`이 잘못됐거나 폐기됨.
* 봇 토큰이 rotate됐는데 secrets가 업데이트 안 된 케이스가 많음.

### `400 Bad Request: chat not found`

* `chat_id` 오타 또는 봇이 해당 채팅방에 참여/권한이 없음.
* 그룹 채팅은 봇이 초대되어 있어야 하고, 경우에 따라 메시지 권한이 필요.

### 메시지에 마크다운/HTML이 깨짐

* 이 액션은 `parse_mode`를 설정하지 않음(Plain text).
* 마크다운 렌더링이 필요하면 `parse_mode=MarkdownV2` 등을 추가하는 별도 액션/옵션화가 필요.
