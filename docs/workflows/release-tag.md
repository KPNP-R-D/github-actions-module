# Release Tag (workflow_dispatch)

리포지토리의 `vX.Y.Z` 태그를 기준으로 **다음 버전 태그를 생성**하고, 동일 메이저의 **`vX` 태그를 최신 버전으로 강제 이동**한다. 옵션으로 GitHub Release도 생성한다.

> 이 워크플로는 “Reusable”이 아니라 **수동 실행(workflow_dispatch)** 전용이다.

---

## Contract

### Required permissions

Workflow에서 선언됨:

```yaml
permissions:
  contents: write
```

* annotated tag 생성/푸시, major tag 강제 업데이트(Force push), GitHub Release 생성에 필요

### Required checkout

* `actions/checkout@v4` with `fetch-depth: 0`

  * 버전 계산을 위해 태그 전체가 필요 (`git tag --list ...`)

### Concurrency

```yaml
concurrency:
  group: ${{ github.workflow }}
  cancel-in-progress: false
```

* 동일 워크플로 이름 기준으로 동시에 2개 실행되지 않게 직렬화
* 진행 중 실행을 취소하지 않음(태그 생성 작업 안정성 우선)

---

## Inputs (workflow_dispatch)

| Input                 | Type   | Default | Description                               |
| --------------------- | ------ | ------: | ----------------------------------------- |
| `bump`                | choice | `patch` | 자동 증가 단위 `major/minor/patch`              |
| `version_override`    | string |    `""` | 수동 지정 버전. `vX.Y.Z` 형식만 허용. 값이 있으면 bump 무시 |
| `release_notes`       | string |    `""` | 태그 메시지 및 Release 본문에 포함(선택)               |
| `make_github_release` | choice | `false` | `"true"`면 GitHub Release 생성               |

---

## Outputs

* 공식 workflow output은 없음
* 실행 로그/summary로 다음을 제공:

  * `last` (이전 태그)
  * `next` (생성한 태그)
  * `major` (`vX`)

---

## Versioning rules

### Tag discovery

* 최신 태그 탐색:

  * `git tag --list "v[0-9]*.[0-9]*.[0-9]*" --sort=-v:refname | head -n1`
* 태그가 없으면 `v0.0.0`을 기준으로 시작

### Next version resolution

1. `version_override`가 있으면:

* 정규식 `^v[0-9]+\.[0-9]+\.[0-9]+$` 검증 통과 시 그대로 사용

2. 없으면 bump로 자동 증가:

* major: `MA+1`, `MI=0`, `PA=0`
* minor: `MI+1`, `PA=0`
* patch: `PA+1`

### Major tag policy

* `vX` 태그는 항상 가장 최신 `vX.Y.Z`를 가리키도록 강제 이동
* 구현:

  * `git tag -fa "${MAJOR}" ...`
  * `git push origin "${MAJOR}" --force`

---

## What it does

### Side effects

* `vX.Y.Z` annotated tag 생성 및 push
* `vX` annotated tag를 `vX.Y.Z`로 이동시키고 force push
* 옵션으로 GitHub Release 생성(이미 존재하면 스킵)

---

<details>
<summary><strong>Steps detail</strong></summary>

### 0) Checkout (with tags)

* `fetch-depth: 0`

### 1) Git user config

* `github-actions[bot]` identity 설정 (annotated tag message 작성 시 사용)

### 2) Resolve next version

* `LAST_TAG` 계산
* `NEXT` 계산(override 또는 bump)
* `MAJOR` 계산(예: NEXT가 v2.3.4면 MAJOR=v2)
* step outputs:

  * `last`, `next`, `major`

### 3) Create annotated tag

* `git tag -a "${NEXT}" -m "Release ${NEXT}\n\n${NOTES}"`
* `git push origin "${NEXT}"`

### 4) Move major tag (vX -> NEXT)

* `git tag -fa "${MAJOR}" -m "Update ${MAJOR} to ${NEXT}"`
* `git push origin "${MAJOR}" --force`

### 5) Create GitHub Release (optional)

* 조건: `inputs.make_github_release == 'true'`
* `gh release view "${NEXT}"`로 존재 확인 후 없으면 생성
* `GH_TOKEN: ${{ github.token }}` 사용

### 6) Summary

* last/next/major 출력

</details>

---

## Usage notes (운영 기준)

* `vX` 강제 이동을 기본 정책으로 택했기 때문에, **`@v1` 같은 “메이저 트래킹 태그”를 실제로 운영**하려는 레포에 적합함.
* 반대로, `vX`를 고정하고 싶거나 메이저 트래킹을 금지하는 레포라면 이 워크플로는 정책 불일치.

---

## Troubleshooting

### `version_override는 vX.Y.Z 형식이어야 합니다`

* `v1.2.3`만 허용. `1.2.3`, `v1.2`, `v1.2.3-rc1` 등은 실패.

### `git push origin vX --force`가 실패

* 브랜치 보호/태그 보호 규칙(Protected tags) 때문에 force push가 차단된 케이스.
* GitHub 설정에서 `v*` 보호 규칙 확인 필요(특히 `v1` 같은 메이저 태그를 보호해두면 항상 실패).

### `gh: not found` 또는 GitHub Release 생성 실패

* `ubuntu-latest`에는 보통 `gh`가 있지만, 러너/환경에 따라 미설치일 수 있음.
* 실패 시:

  * `actions/setup-gh`(또는 패키지 설치) 단계 추가 고려
  * `GH_TOKEN` scope/권한(contents:write) 확인

### 동시 실행으로 태그 충돌 가능

* concurrency로 직렬화되지만, 다른 워크플로/사용자가 병행으로 태그를 만들면 충돌 가능.
* 강하게 막으려면 “태그 존재 여부 재검증” 또는 “원격 최신 태그 fetch 후 다시 계산” 로직을 강화해야 함.
