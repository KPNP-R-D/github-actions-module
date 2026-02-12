# Publish Tags And Release (Composite Action)

`{project}/vX.Y.Z` **annotated release tag**를 생성/푸시하고, `{project}/prod` 태그를 해당 릴리스 태그로 **이동(필요 시 force push)** 한다. 옵션으로 `next_tag`에 대한 **GitHub Release**까지 생성한다.

---

## Contract

### Required permissions

이 액션은 `git push`와(옵션) `gh release create`를 수행한다. 따라서 caller workflow에서 다음 권한이 필요하다.

* `permissions: contents: write` (필수)

  * 태그 생성/푸시, prod 태그 이동(force push 포함)
* GitHub Release 생성까지 하면:

  * `GH_TOKEN`(보통 `${{ github.token }}` 또는 PAT)을 env로 제공해야 함
  * `github.token`을 쓸 경우에도 `contents: write`가 필요

### Required checkout

* 액션 내부에서 `actions/checkout@v4` with `fetch-depth: 0`

  * 태그 조회/생성/이동을 위해 태그 전체 필요

### Prerequisites

* `git` 및 `gh` CLI가 runner에 있어야 함

  * GitHub-hosted runner(ubuntu-latest)에서는 일반적으로 충족
* 보호 태그(Protected tags) 정책이 있으면 force push가 차단될 수 있음

---

## Inputs

### Required

| Input      | Type   | Description                                          |
| ---------- | ------ | ---------------------------------------------------- |
| `project`  | string | 태그 prefix. 예: `admin` → `admin/v1.2.3`, `admin/prod` |
| `next_tag` | string | 생성할 릴리스 태그(Full). 예: `admin/v1.2.3`                  |
| `prod_tag` | string | 이동할 prod 태그(Full). 예: `admin/prod`                   |

### Optional

| Input                             | Type   |                        Default | Description                                           |
| --------------------------------- | ------ | -----------------------------: | ----------------------------------------------------- |
| `release_notes`                   | string |                           `""` | annotated tag 메시지 및 Release 본문에 포함(선택)                |
| `make_github_release`             | string |                        `false` | `"true"`면 `gh release create` 수행 (GH_TOKEN 필요)        |
| `allow_move_existing_release_tag` | string |                        `false` | `"true"`면 이미 존재하는 `next_tag`를 overwrite/force move 허용 |
| `force_update_prod_tag`           | string |                         `true` | `"true"`면 `prod_tag` 업데이트를 force push (권장 true)       |
| `git_user_name`                   | string |          `github-actions[bot]` | annotated tag 생성용 git user.name                       |
| `git_user_email`                  | string | `...@users.noreply.github.com` | annotated tag 생성용 git user.email                      |

---

## Outputs

| Output            | Description                                                              |
| ----------------- | ------------------------------------------------------------------------ |
| `next_tag`        | 입력 `next_tag` echo                                                       |
| `prod_tag`        | 입력 `prod_tag` echo                                                       |
| `release_created` | `make_github_release`가 `"true"`면 `true`, 아니면 `false` (실제 생성 여부를 조회하진 않음) |

---

## Validation rules (strict)

이 액션은 입력 형식을 강하게 고정한다.

* `next_tag`는 반드시 아래 정규식을 만족해야 함:

  * `^${project}/v[0-9]+\.[0-9]+\.[0-9]+$`

  예: `admin/v1.2.3` ✅
  예: `admin/v1.2` ❌, `admin/1.2.3` ❌, `v1.2.3` ❌

* `prod_tag`는 반드시 아래와 정확히 일치해야 함:

  * `${project}/prod`

---

## What it does

### Side effects

1. 태그 fetch
2. `next_tag` annotated tag 생성 또는(옵션) overwrite 후 push
3. `prod_tag` annotated tag를 `next_tag`로 이동 후 push (기본 force)
4. (옵션) `next_tag`에 대한 GitHub Release 생성(존재하면 스킵)

---

<details>
<summary><strong>Steps detail</strong></summary>

### 0) Checkout (with tags)

* `actions/checkout@v4`, `fetch-depth: 0`

### 1) Git user config

* `inputs.git_user_name/email`로 설정

### 2) Validate inputs

* `project/next_tag/prod_tag` 비어있으면 실패
* `next_tag` 포맷 검증
* `prod_tag == "${project}/prod"` 강제

### 3) Fetch tags

* `git fetch --tags --force`

### 4) Create or move release tag (annotated) and push

* 메시지:

  * 기본 `Release ${NEXT}`
  * `release_notes`가 있으면 본문에 추가

* 동작:

  * `refs/tags/${NEXT}` 존재 시:

    * `allow_move_existing_release_tag=true`면 `git tag -fa`로 overwrite
    * 아니면 실패
  * 원격 push:

    * allow_move=true면 `git push origin "${NEXT}" --force`
    * 아니면 `git push origin "${NEXT}"`

### 5) Move prod tag to release tag and push

* 항상 로컬에서 `git tag -fa "${PROD}" -m "Update ..."`
* push:

  * `force_update_prod_tag=true`면 `--force`
  * 아니면 일반 push

### 6) Create GitHub Release (optional)

* 조건: `make_github_release == 'true'`
* 요구: `GH_TOKEN`이 **env로 존재**해야 함 (`env.GH_TOKEN`)
* `gh release view "${NEXT}"`로 존재 여부 확인 후 없으면 생성

### 7) Outputs

* next_tag/prod_tag echo
* release_created는 make_github_release 입력값 기준으로만 설정

</details>

---

## Caller example

### 1) 태그 생성 + prod 이동 (Release 생성 안 함)

```yaml id="y9kfl5"
permissions:
  contents: write

jobs:
  publish_tags:
    runs-on: ubuntu-latest
    steps:
      - name: Publish tags
        uses: 5n-project/github-actions-module/actions/publish-tags-and-release@v1
        with:
          project: admin
          next_tag: admin/v1.2.3
          prod_tag: admin/prod
          force_update_prod_tag: "true"
```

### 2) GitHub Release까지 생성

```yaml id="eexsci"
permissions:
  contents: write

jobs:
  publish_tags:
    runs-on: ubuntu-latest
    steps:
      - name: Publish tags + GitHub Release
        uses: 5n-project/github-actions-module/actions/publish-tags-and-release@v1
        env:
          GH_TOKEN: ${{ github.token }}
        with:
          project: admin
          next_tag: admin/v1.2.3
          prod_tag: admin/prod
          release_notes: |
            - Fix: 로그인 토큰 갱신 버그 수정
            - Feat: 배치 모니터링 추가
          make_github_release: "true"
          allow_move_existing_release_tag: "false"
          force_update_prod_tag: "true"
```

---

## Troubleshooting

### `next_tag must be '{project}/vX.Y.Z'`

* `project`와 `next_tag` prefix가 불일치한 케이스가 대부분.

  * `project=admin`이면 `next_tag=admin/v1.2.3`만 통과.

### `prod_tag must be exactly '{project}/prod'`

* prod 태그 정책이 고정이라 `admin/production` 같은 변형은 불가.
* 환경별 태그가 필요하면 `prod_tag` 룰을 완화하거나 입력을 `env_tag` 같은 일반화로 바꿔야 함.

### `Tag ... already exists. Refusing to overwrite`

* 같은 `next_tag`를 재발행하려는 상황.
* 의도적으로 덮어쓸 때만 `allow_move_existing_release_tag=true`를 켜야 함(기본 false가 안전).

### `git push ... --force` 실패

* Protected tags / 브랜치 보호 정책으로 force push가 차단될 수 있음.
* `force_update_prod_tag=false`로 바꾸면 일반 push로 시도하지만, prod 태그가 이미 존재하면 업데이트 자체가 실패할 수 있음(정책과 목적이 충돌).

### `GH_TOKEN env is required when make_github_release=true`

* `env: GH_TOKEN: ...`를 step 또는 job 레벨에서 반드시 주입해야 함.
* `with:`가 아니라 `env:`로 받는 구조라는 점이 포인트.

### `release_created`가 true인데 실제로는 “이미 존재해서 스킵”

* output은 “생성 시도 모드”를 의미하고, 실제 생성 여부를 조회하지 않음.
* 정확한 결과가 필요하면:

  * `gh release view` 결과에 따라 output을 세팅하도록 수정해야 함.
