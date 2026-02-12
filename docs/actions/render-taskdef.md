# Render ECS Task Definition (Composite Action)

ECS task definition **템플릿 JSON**을 `envsubst`로 렌더링하고, 옵션으로 `.env` 파일을 읽어 `containerDefinitions[].environment`에 주입할 수 있도록 **JSON 배열 형태(`ENVIRONMENT_JSON`)로 변환**한다. 렌더 후에는 **미치환 placeholder를 검증**하고, 로그 노출을 줄이기 위해 **sanitized preview**를 출력한다.

---

## Contract

### Required permissions

* GitHub 권한은 필요 없음(외부 API/태그/릴리스 작업 없음)
* 단, `apt-get`을 사용하므로 GitHub-hosted runner(ubuntu-latest) 기준으로 동작을 전제

### Runtime dependencies

이 액션이 runner에 설치/사용하는 도구:

* `gettext-base` (`envsubst`)
* `jq`

> 액션 내부에서 `sudo apt-get install` 수행.

### Template requirements

* `template_path`는 유효한 JSON이어야 하고, `envsubst` 치환 후에도 JSON이 유지돼야 함.
* 미치환 placeholder는 실패 처리(아래 규칙 참고).

---

## Inputs

### Required

| Input           | Type   | Description         |
| --------------- | ------ | ------------------- |
| `template_path` | string | taskdef 템플릿 JSON 경로 |

### Optional

| Input           | Type   |        Default | Description                                            |
| --------------- | ------ | -------------: | ------------------------------------------------------ |
| `output_path`   | string | `taskdef.json` | 렌더 결과 파일 경로                                            |
| `env_file_path` | string |           `""` | `.env` 파일 경로. `KEY=VALUE` 라인들을 `ENVIRONMENT_JSON`으로 변환 |

---

## Outputs

| Output         | Description                       |
| -------------- | --------------------------------- |
| `taskdef_path` | 렌더된 taskdef 파일 경로 (`output_path`) |

---

## Env injection spec (`env_file_path`)

### Supported format

* 각 라인: `KEY=VALUE`
* 무시 규칙:

  * 빈 줄
  * `#`로 시작하는 주석 라인
* KEY validation:

  * 정규식 `^[A-Za-z_][A-Za-z0-9_]*$`
* VALUE:

  * 첫 번째 `=` 기준으로 split
  * trim 수행(양쪽 공백 제거)
  * `=` 포함 값도 허용

### Result

`.env` → ECS environment 배열(JSON):

```json
[
  { "name": "KEY1", "value": "VALUE1" },
  { "name": "KEY2", "value": "VALUE2" }
]
```

이 JSON 배열은 **환경변수 `ENVIRONMENT_JSON`** 으로 export된다.

> 중요: 이 액션 자체는 taskdef JSON의 `containerDefinitions[].environment`를 직접 수정하지 않는다.
> 대신 템플릿에서 `${ENVIRONMENT_JSON}` 같은 placeholder로 **주입되도록 설계**된 구조다.

---

## Placeholder rendering + validation

### Rendering

* `envsubst < TEMPLATE | jq . > OUT`

  * `jq .`로 JSON 파싱이 되지 않으면 step 실패

### Unreplaced placeholders check

렌더 결과(`OUT`)에 아래 패턴이 남아있으면 실패:

* `\${[A-Za-z_][A-Za-z0-9_]*}`

즉 `${SOME_VAR}`가 남아있으면 **치환 누락으로 간주**하고 차단한다.

---

## What it does

### Side effects

* 렌더 결과 파일 생성: `${output_path}`
* 로그 출력용 sanitize 파일 생성: `taskdef.sanitized.json`
* sanitize된 JSON을 stdout에 출력(cat)

### Sanitized preview policy

* `.containerDefinitions[].secrets` 제거
* `.containerDefinitions[].environment`에서 name이 아래 키워드를 포함하면 제거(대소문자 무시):

  * `PASSWORD|SECRET|KEY|TOKEN`

> sanitize는 “로그 노출 감소” 목적이며, 실제 OUT 파일에는 영향을 주지 않는다.

---

<details>
<summary><strong>Steps detail</strong></summary>

### 0) Install envsubst & jq

* `sudo apt-get update`
* `sudo apt-get install gettext-base jq`

### 1) Render task definition (envsubst + env file injection)

1. TEMPLATE 파일 존재 검증
2. `.env` 파일이 지정된 경우:

   * 파일 존재 검증
   * 라인 파싱/검증 후 JSON 배열 생성 → `ENVIRONMENT_JSON` export
3. `envsubst`로 템플릿 렌더 → `jq`로 JSON 검증 → OUT 저장
4. OUT에서 미치환 placeholder 검사(남아있으면 실패)
5. sanitized JSON 생성 및 출력
6. output: `taskdef_path=${OUT}`

</details>

---

## Template example (ENVIRONMENT_JSON 주입 패턴)

템플릿 JSON에서 아래처럼 사용해야 실제로 주입된다.

```json
{
  "containerDefinitions": [
    {
      "name": "app",
      "environment": ${ENVIRONMENT_JSON}
    }
  ]
}
```

> `ENVIRONMENT_JSON`는 JSON 배열 문자열이므로, 따옴표로 감싸면 JSON이 깨짐. (즉 `"${ENVIRONMENT_JSON}"` ❌)

---

## Caller example

```yaml id="osim7g"
- name: Render taskdef
  id: taskdef
  uses: 5n-project/github-actions-module/actions/render-ecs-taskdef@v1
  env:
    # envsubst 대상 변수들 (템플릿에서 ${VAR}로 사용)
    AWS_REGION: ap-northeast-2
    TASK_CPU: "512"
    TASK_MEMORY: "1024"
  with:
    template_path: infra/ecs/taskdef.template.json
    output_path: taskdef.json
    env_file_path: kpnp-message/broker.env

- name: Upload taskdef artifact
  uses: actions/upload-artifact@v4
  with:
    name: taskdef
    path: ${{ steps.taskdef.outputs.taskdef_path }}
```

---

## Troubleshooting

### `Template not found: ...`

* 경로 오타 또는 checkout/working-directory 문제.
* 템플릿이 caller repo에 있는지, 상대경로 기준이 runner workspace인지 확인.

### `Env file not found: ...`

* `env_file_path`가 빈 값이 아닌데 파일이 없음.
* 환경별 파일을 쓰면 (prod/dev) caller에서 조건 분기해서 넘기는 게 안전.

### `Invalid env line (missing '=')`

* `.env`에 `KEY VALUE` 같은 라인이 섞여 있음.
* 주석은 `#`로 시작해야 무시됨.

### `Invalid env key: ...`

* KEY에 `-` 포함, 시작이 숫자, 공백 등 규칙 위반.
* ECS 환경변수명 컨벤션에 맞게 정리 필요.

### `parse error: Invalid numeric literal` 등 jq 파싱 실패

* envsubst 결과가 JSON이 깨짐.
* 흔한 원인:

  * JSON에 문자열이 들어가야 하는데 따옴표 없이 치환됨
  * `${ENVIRONMENT_JSON}`를 따옴표로 감싸서 JSON 배열이 문자열로 들어감
  * 값에 개행/따옴표가 들어가 escaping이 필요

### `Unreplaced placeholders remain`

* 템플릿에 `${VAR}`가 남아있는데 runner env에 `VAR`가 정의되지 않음.
* caller에서 `env:`로 필요한 변수를 모두 export해야 함.

### 민감정보가 로그에 노출될까?

* sanitized preview에서 secrets는 제거하지만,

  * environment 키 이름이 `PASSWORD/SECRET/KEY/TOKEN` 패턴에 안 걸리면 그대로 출력될 수 있음.
* 더 강한 보안이 필요하면:

  * preview 출력 자체를 끄거나
  * allowlist 기반으로 environment를 출력하도록 정책 변경 권장.
