# resolve-release-version (Composite Action)

프로젝트별 태그 네이밍 규칙 `{project}/vX.Y.Z`를 기준으로, 최신 태그를 찾고 다음 릴리스 버전을 **자동 증가(bump)** 또는 **override**로 결정한다. 결과로 `last_tag`, `next_tag`, `major_tag({project}/vX)`, `version(vX.Y.Z)`를 output으로 제공한다.

---

## Contract

### Required permissions

* 기본적으로 repo를 checkout하고 `git tag`를 조회한다.
* 권장:

  * `permissions: contents: read`
* 태그를 생성/푸시하지는 않으므로 `contents: write`는 불필요.

### Required checkout

* 액션 내부에서 `actions/checkout@v4` with `fetch-depth: 0`

  * 태그 목록 조회 및 버전 계산에 필요

### Tag format assumptions

* 프로젝트 태그는 **반드시** `{project}/vX.Y.Z` 형식이어야 함.

  * 예: `admin/v1.2.3`, `kpnp-champs/v2.0.1`

---

## Inputs

### Required

| Input     | Type   | Description           |
| --------- | ------ | --------------------- |
| `project` | string | 태그 prefix. 예: `admin` |

### Optional

| Input                  | Type   |  Default | Description                                  |       |        |
| ---------------------- | ------ | -------: | -------------------------------------------- | ----- | ------ |
| `bump`                 | string |  `patch` | `version_override`가 비어있을 때 증가 단위 `major      | minor | patch` |
| `version_override`     | string |     `""` | 직접 지정 버전 `vX.Y.Z`. 값이 있으면 bump 무시            |       |        |
| `default_last_version` | string | `v0.0.0` | 해당 프로젝트 태그가 하나도 없을 때 기준 last version         |       |        |
| `fetch_tags`           | string |   `true` | `"true"`면 `git fetch --tags --force` 수행 후 계산 |       |        |

---

## Outputs

| Output      | Description                  |
| ----------- | ---------------------------- |
| `last_tag`  | 최신 존재 태그. 예: `admin/v1.2.3`  |
| `next_tag`  | 다음 릴리스 태그. 예: `admin/v1.2.4` |
| `major_tag` | 메이저 트래킹 태그. 예: `admin/v1`    |
| `project`   | 입력 project echo              |
| `version`   | `vX.Y.Z`만 반환(태그 prefix 제외)   |

---

## Version resolution rules

### 1) Latest tag discovery

* 조회 패턴: `${project}/v[0-9]*.[0-9]*.[0-9]*`
* 정렬: `--sort=-v:refname` (버전 내림차순)
* 최신 1개를 `LAST_TAG`로 선택

태그가 없으면:

* `default_last_version`를 검증(`vX.Y.Z`) 후
* `LAST_TAG="${project}/${default_last_version}"`로 가정

### 2) Next version 결정

* `version_override`가 있으면:

  * `^v[0-9]+\.[0-9]+\.[0-9]+$` 검증 후 그대로 사용
* 없으면 bump 적용:

  * major: `MA+1`, `MI=0`, `PA=0`
  * minor: `MI+1`, `PA=0`
  * patch: `PA+1`

### 3) next_tag / major_tag 생성

* `next_tag = {project}/{NEXT_VER}`
* `major_tag = {project}/vX` (NEXT_VER 기준)

---

<details>
<summary><strong>Steps detail</strong></summary>

### 0) Checkout

* `actions/checkout@v4` with `fetch-depth: 0`

### 1) Fetch tags (optional)

* 조건: `inputs.fetch_tags == 'true'`
* 실행: `git fetch --tags --force`

### 2) Resolve next version for project

* project/override/bump/default_last_version 읽기
* latest tag 탐색 및 fallback 처리
* next version 계산
* outputs write:

  * `project`, `last`, `next`, `major`, `version`

</details>

---

## Caller examples

### 1) 기본 patch 증가

```yaml id="xyw7ak"
- name: Resolve next version
  id: ver
  uses: 5n-project/github-actions-module/actions/resolve-release-version@v1
  with:
    project: admin
    bump: patch

- name: Print
  run: |
    echo "last = ${{ steps.ver.outputs.last_tag }}"
    echo "next = ${{ steps.ver.outputs.next_tag }}"
    echo "major= ${{ steps.ver.outputs.major_tag }}"
    echo "ver  = ${{ steps.ver.outputs.version }}"
```

### 2) override 사용

```yaml id="t07sj1"
- name: Resolve next version (override)
  id: ver
  uses: 5n-project/github-actions-module/actions/resolve-release-version@v1
  with:
    project: kpnp-champs
    version_override: v2.1.0
```

### 3) 태그 fetch 생략 (로컬 태그가 이미 최신인 파이프라인에서)

```yaml id="nq6m0r"
- name: Resolve next version (no fetch)
  id: ver
  uses: 5n-project/github-actions-module/actions/resolve-release-version@v1
  with:
    project: admin
    fetch_tags: "false"
```

---

## Troubleshooting

### `project is required`

* 입력 누락. reusable 호출에서 `with: project: ...` 전달 확인.

### `default_last_version must be vX.Y.Z`

* `v0.0.0` 형식만 허용.
* `0.0.0` 또는 `v0.0` 같은 값은 실패.

### `Last tag '{...}' is not '{project}/vX.Y.Z'`

* 해당 project prefix 아래에 규칙을 깨는 태그가 “최신”으로 잡힌 경우.
* 해결:

  * 잘못된 태그 정리(삭제/이름 변경)
  * 태그 패턴을 더 엄격하게(현재도 꽤 엄격하지만, 예외 케이스가 섞이면 최신 정렬 결과가 꼬일 수 있음)

### `version_override must be vX.Y.Z`

* `v1.2.3`만 허용. prerelease(`-rc1`) 불가.

### `bump must be one of major|minor|patch`

* bump 입력 값 오타/대소문자 이슈.
* caller에서 workflow_dispatch choice로 강제하는 게 안전.
