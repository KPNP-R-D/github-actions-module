## 1) `docs/actions/gemini-generate.md`


# Gemini Generate (Composite Action)

Gemini `generateContent` API를 호출해서 **생성된 text를 GitHub Actions output(`text`)으로 반환**한다.

- Endpoint: `https://generativelanguage.googleapis.com/v1beta/models/<MODEL>:generateContent`
- Auth: `x-goog-api-key` 헤더

---

## Contract

### Requires
- runner에 `jq` 필요 (GitHub-hosted ubuntu-latest에는 기본 존재)
- `curl` 필요 (기본 존재)

### Outputs
- `text`
  - `steps.<id>.outputs.text` 로 사용

---

## Inputs

### `api_key` (required)
Gemini API Key (GitHub Secrets로 전달)

### `model` (required)
모델 ID
- 예: `gemini-2.5-flash`

### `content` (required)
유저 프롬프트 본문(최종 입력 텍스트)

### `temperature` (optional, default: `"0.1"`)
generationConfig.temperature  
- action 내부에서 `--argjson`으로 주입되므로 **숫자 문자열**이어야 한다.
  - 예: `"0.3"`

### `max_output_tokens` (optional, default: `"2048"`)
generationConfig.maxOutputTokens  
- action 내부에서 `--argjson`으로 주입되므로 **정수 문자열**이어야 한다.
  - 예: `"50000"`

### `system_instruction` (optional, default: `""`)
system instruction 텍스트.
- 비어있지 않으면 request body에 `systemInstruction.parts[0].text`로 포함된다.

---

## Request payload behavior

### system_instruction이 있는 경우
```json
{
  "systemInstruction": { "parts": [ { "text": "<system_instruction>" } ] },
  "contents": [ { "role": "user", "parts": [ { "text": "<content>" } ] } ],
  "generationConfig": {
    "temperature": <temperature>,
    "maxOutputTokens": <max_output_tokens>
  }
}
````

### system_instruction이 비어있는 경우

```json
{
  "contents": [ { "role": "user", "parts": [ { "text": "<content>" } ] } ],
  "generationConfig": {
    "temperature": <temperature>,
    "maxOutputTokens": <max_output_tokens>
  }
}
```

---

## Error handling

### 1) API error

응답에 `.error`가 존재하면 실패 처리:

* 로그: `::error::Gemini API error: <error-json>`
* exit 1

### 2) Empty text

`.candidates[0].content.parts[0].text`가 비어있으면 실패:

* 로그: `::error::Gemini returned empty text. Raw response: <raw>`
* exit 1

---

## Outputs (GITHUB_OUTPUT)

멀티라인 안전을 위해 랜덤 delimiter로 `text` output을 기록한다.

```bash
DELIM="__EOF_$(date +%s%N)__"
{
  echo "text<<$DELIM"
  echo "$TEXT"
  echo "$DELIM"
} >> "$GITHUB_OUTPUT"
```

---

## Usage

### Basic

```yaml
- name: Gemini generate
  id: llm
  uses: 5n-project/github-actions-module/actions/gemini-generate@gemini-generate/v1
  with:
    api_key: ${{ secrets.GEMINI_API_KEY }}
    model: gemini-2.5-flash
    content: ${{ steps.prompt.outputs.content }}
```

### With system instruction + tuning

```yaml
- name: Gemini generate
  id: llm
  uses: 5n-project/github-actions-module/actions/gemini-generate@gemini-generate/v1
  with:
    api_key: ${{ secrets.GEMINI_API_KEY }}
    model: gemini-2.5-flash
    system_instruction: ${{ steps.prompt.outputs.prompt }}
    content: ${{ steps.prompt.outputs.content }}
    temperature: "0.3"
    max_output_tokens: "50000"
```

결과 사용:

```yaml
- name: Use output
  run: echo "${{ steps.llm.outputs.text }}"
```

---

## Notes / Pitfalls

* `temperature`, `max_output_tokens`는 jq `--argjson`으로 넣기 때문에 **숫자여야 함**.

  * `"abc"` 같은 값이면 jq 단계에서 실패한다.
* 응답 파싱은 현재 단일 후보/단일 파트 기준:

  * `.candidates[0].content.parts[0].text`
  * 모델/응답 포맷이 바뀌면 보강 필요.
