# Build & Push to ECR (Reusable)

지정한 `workdir` 기준으로 **Dockerfile 또는 docker-compose(프로필별)** 를 감지해서 이미지를 빌드하고 **AWS ECR로 push**한다. 멀티플랫폼(`buildx`) + GHA cache를 기본 사용한다.

---

## Contract

### Required permissions

Reusable workflow 내부에서 이미 선언됨:

```yaml
permissions:
  contents: read
  id-token: write
```

* `id-token: write`: AWS OIDC AssumeRole에 필요
* `contents: read`: checkout에 필요

### Required checkout

* 내부에서 `actions/checkout@v4` 수행 (`fetch-depth: 0`)

  * compose가 태그/히스토리를 필요로 하지는 않지만, 레포 전체 컨텍스트 빌드 안정성을 위해 전체 히스토리로 고정되어 있음

### Prerequisites

* Caller repo에는 아래 중 하나가 존재해야 함:

  1. `${workdir}/Dockerfile`
  2. `${workdir}/docker-compose.<spring_profiles_active>.yml` 또는 `${workdir}/docker-compose.yml`

* AWS IAM Role(AssumeRole 대상)에 최소 권한 필요:

  * ECR push 권한 (ecr:BatchCheckLayerAvailability, ecr:InitiateLayerUpload, ecr:UploadLayerPart, ecr:CompleteLayerUpload, ecr:PutImage 등)
  * (대부분 `AmazonEC2ContainerRegistryPowerUser` 또는 세분화된 커스텀 정책으로 대체)

---

## workflow_call inputs

### Required

| Input              | Type   | Description                                                                              |
| ------------------ | ------ | ---------------------------------------------------------------------------------------- |
| `workdir`          | string | 빌드 대상 디렉토리(리포 루트 기준). 예: `kpnp-message`                                                  |
| `ecr_repo_uri`     | string | ECR repo URI (registry/repo). 예: `1234.dkr.ecr.ap-northeast-2.amazonaws.com/deploy-test` |
| `image_tag`        | string | 이미지 태그. 예: `v1.2.3`                                                                      |
| `aws_region`       | string | AWS 리전. 예: `ap-northeast-2`                                                              |
| `environment_name` | string | GitHub Environment 이름. job에 `environment`로 연결됨                                           |

### Optional

| Input                    | Type   |       Default | Description                                                      |
| ------------------------ | ------ | ------------: | ---------------------------------------------------------------- |
| `platforms`              | string | `linux/amd64` | buildx 플랫폼. 예: `linux/amd64,linux/arm64`                         |
| `spring_profiles_active` | string |        `prod` | 프로필 값. compose 파일 탐색 및 build arg로 전달됨                            |
| `service`                | string |          `""` | compose bake target 제한용. 비면 전체, 값이 있으면 `,` 또는 공백 구분으로 targets 지정 |

---

## workflow_call outputs

| Output      | Description                                             |
| ----------- | ------------------------------------------------------- |
| `image_ref` | `repo:tag` 형태. 예: `...amazonaws.com/deploy-test:v1.2.3` |

---

## workflow_call secrets

Required:

* `AWS_ROLE_ARN`

  * `aws-actions/configure-aws-credentials@v4`의 `role-to-assume`로 사용

---

## Build strategy (deterministic rules)

빌드 전략은 아래 순서로 결정된다.

1. **`${workdir}/Dockerfile` 존재**

   * `docker buildx build --push`
   * 빌드 컨텍스트: **repo root (`${ROOT}`)**
     (멀티모듈/공용 모듈 참조를 위해 root 컨텍스트 고정)

2. **Dockerfile 없음 → Compose 감지**

   * `${workdir}`로 들어가서 compose 파일 선택:

     * 우선순위: `docker-compose.<profile>.yml` → `docker-compose.yml`
   * `docker buildx bake --push`
   * tags/platform/cache/build-arg를 `--set`으로 주입

---

## What it does

### Outputs / Side effects

* ECR에 `ecr_repo_uri:image_tag`로 이미지 push
* Step Summary에는 별도 출력 없음(로그로 정보 출력)
* workflow output으로 `image_ref` 제공

### Caching

* GHA cache 사용:

  * `--cache-from type=gha`
  * `--cache-to type=gha,mode=max`

---

<details>
<summary><strong>Steps detail</strong></summary>

### 0) Checkout

* `actions/checkout@v4` with `fetch-depth: 0`

### 1) Set up Buildx

* `docker/setup-buildx-action@v3`
* driver: `docker-container`

### 2) Configure AWS credentials (OIDC)

* `aws-actions/configure-aws-credentials@v4`
* `secrets.AWS_ROLE_ARN`로 AssumeRole

### 3) Login to ECR

* `aws-actions/amazon-ecr-login@v2`

### 4) Build & Push image

공통:

* 입력값 trim/normalize (`\r\n` 제거 + xargs)
* `IMAGE="${ECR_REPO_URI}:${IMAGE_TAG}"`로 고정

#### 4-1) Dockerfile path exists → buildx build

* 경로: `${ROOT}/${WORKDIR}/Dockerfile`
* 실행:

  * `--platform "${PLATFORMS}"`
  * `--build-arg SPRING_PROFILES_ACTIVE="${PROFILE}"`
  * `-t "${IMAGE}"`
  * `-f "${DOCKERFILE}"`
  * build context: `"${ROOT}"`
  * `--push`

#### 4-2) Compose → buildx bake

* compose 파일 선택:

  * `docker-compose.${PROFILE}.yml` 있으면 우선 사용
  * 없으면 `docker-compose.yml`
  * 둘 다 없으면 실패

* entitlement 자동 허용(휴리스틱):

  * compose 내 `context: ..` 감지 시 `--allow=fs.read=..`
  * `context: ../..` 감지 시 `--allow=fs.read=../..`

* bake 실행:

  * `--file "${COMPOSE_ABS}"`
  * targets: `service` 지정 시 해당 targets만, 아니면 전체
  * tags/platform/cache/build-arg를 `--set`으로 강제 주입
  * `--push`

### 5) Output image ref

* `GITHUB_OUTPUT`에 `image_ref=<repo:tag>` 기록

</details>

---

## Caller example

```yaml
jobs:
  build_push:
    uses: 5n-project/github-actions-module/.github/workflows/build-and-push-to-ecr.reusable.yml@v1.0.0
    with:
      workdir: kpnp-message
      ecr_repo_uri: 123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/deploy-test
      image_tag: v1.2.3
      aws_region: ap-northeast-2
      environment_name: deploy-test-prod

      platforms: linux/amd64
      spring_profiles_active: prod
      service: app,broker
    secrets:
      AWS_ROLE_ARN: ${{ secrets.AWS_ROLE_ARN }}
```

---

## Troubleshooting

### `::error::workdir is required` / `ecr_repo_uri is required` / `image_tag is required`

* 필수 input 누락.
* `with:`에 값이 실제로 들어오는지(특히 reusable 호출에서 inputs forward) 확인.

### `No docker-compose.<profile>.yml and no docker-compose.yml in <workdir>`

* `${workdir}` 경로가 틀렸거나 compose 파일명이 표준과 불일치.
* `spring_profiles_active=prod`면 `docker-compose.prod.yml` 존재 여부부터 확인.

### `resolve : lstat ../<something>: no such file or directory`

* compose 파일 내부 `build.context`가 `..` 등 상위 경로를 참조하는데, bake 실행 위치/경로 해석이 꼬인 케이스.
* 이 워크플로는 **bake를 repo root 컨텍스트로 안정화**하려고 `--file`을 absolute로 넘기지만,
  compose 내부 경로가 더 복잡하면 여전히 깨질 수 있음.

  * 해결: compose의 `context`를 repo-root 기준으로 맞추거나, `workdir` 구조를 단순화.

### `AccessDeniedException ... ecr:DescribeImages` / push 관련 권한 오류

* AssumeRole 대상 IAM policy에 ECR 권한이 부족.
* ECR repo ARN 범위/리전/계정이 맞는지 확인.

### 멀티플랫폼 빌드가 느리거나 실패

* `platforms: linux/amd64,linux/arm64`는 QEMU 에뮬레이션 영향으로 시간이 늘고 실패율이 올라감.
* 필요 없으면 `linux/amd64` 단일로 제한.
