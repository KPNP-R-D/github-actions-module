# Resolve ECR Image Digest (Composite Action)

ECR에서 `{repository}:{image_tag}`에 해당하는 이미지를 조회해 **imageDigest(`sha256:...`)** 를 가져오고, ECS task definition 등에 바로 넣기 좋은 **`repo@sha256:...`** 형태까지 함께 산출한다.

---

## Contract

### Required permissions

이 액션은 `aws ecr describe-images`를 호출한다. 따라서 caller workflow에서 다음이 필요하다.

* `permissions: id-token: write` (OIDC로 AWS AssumeRole 하는 경우)
* AssumeRole 대상 IAM policy에 최소 권한:

  * `ecr:DescribeImages` (필수)
  * (registry-id 지정 시) 해당 registry/repo 접근 범위 포함

> 이 액션 자체에는 `configure-aws-credentials`가 없다.
> 즉, **caller에서 AWS credentials 설정(OIDC)** 을 먼저 해둬야 한다.

### Runtime dependencies

* `jq` 설치 단계가 있지만, 현재 스크립트는 `jq`를 사용하지 않는다.

  * 유지 목적이 아니라면 제거 가능(속도 개선)

---

## Inputs

### Required

| Input            | Type   | Description                                                               |
| ---------------- | ------ | ------------------------------------------------------------------------- |
| `aws_region`     | string | AWS region                                                                |
| `aws_account_id` | string | ECR registry id(계정 ID)                                                    |
| `repository`     | string | ECR repository name (예: `kpnp-champs`)                                    |
| `image_tag`      | string | image tag (예: `v1.2.3`)                                                   |
| `ecr_repo_uri`   | string | full repo URI (예: `123.dkr.ecr.ap-northeast-2.amazonaws.com/kpnp-champs`) |

---

## Outputs

| Output              | Description                 |
| ------------------- | --------------------------- |
| `image_digest`      | `sha256:...`                |
| `image_with_digest` | `<ecr_repo_uri>@sha256:...` |

---

## What it does

* 실행 명령:

```bash id="xgkdx5"
aws ecr describe-images \
  --region "$AWS_REGION" \
  --registry-id "$AWS_ACCOUNT_ID" \
  --repository-name "$REPO" \
  --image-ids imageTag="$TAG" \
  --query 'imageDetails[0].imageDigest' \
  --output text
```

* 결과가 비어있거나 `None`이면 실패 처리.
* 성공 시:

  * `image_digest=sha256:...`
  * `image_with_digest=<repo>@sha256:...`

---

<details>
<summary><strong>Steps detail</strong></summary>

### 0) Install jq

* `sudo apt-get update`
* `sudo apt-get install jq`

> 현재 스크립트에서는 jq 미사용.

### 1) Resolve digest via ECR

* `aws ecr describe-images`로 digest 조회
* output export + notice 로그 출력

</details>

---

## Caller example

### 1) OIDC로 AWS 자격 설정 후 호출

```yaml id="6qn3nt"
permissions:
  id-token: write
  contents: read

jobs:
  resolve_digest:
    runs-on: ubuntu-latest
    steps:
      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ap-northeast-2

      - name: Resolve ECR digest
        id: dig
        uses: 5n-project/github-actions-module/actions/resolve-ecr-image-digest@v1
        with:
          aws_region: ap-northeast-2
          aws_account_id: ${{ secrets.AWS_ACCOUNT_ID }}
          repository: kpnp-champs
          image_tag: v1.2.3
          ecr_repo_uri: 123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/kpnp-champs

      - name: Print
        run: |
          echo "${{ steps.dig.outputs.image_digest }}"
          echo "${{ steps.dig.outputs.image_with_digest }}"
```

### 2) ECS taskdef에 digest pinning

* taskdef 렌더링 시 `${IMAGE}`를 `repo@sha256:...`로 넣으면 “태그 변조” 리스크를 줄일 수 있음.

---

## Troubleshooting

### `Failed to resolve digest from ECR`

* 가장 흔한 원인:

  * `repository` 이름이 URI 경로와 다름 (`kpnp-champs` vs `deploy-test/kpnp-champs` 같은 prefix)
  * `image_tag`가 존재하지 않음
  * 리전/계정 ID 불일치
  * IAM 권한 부족

### `AccessDeniedException ... ecr:DescribeImages`

* AssumeRole policy에 `ecr:DescribeImages`가 없음.
* 리소스 ARN 스코프가 특정 repo로 제한되어 있으면 `repository`와 ARN이 일치해야 함.

### jq 설치가 느림

* jq를 실제로 안 쓰면 설치 단계 제거하는 게 맞음.
* 또는 runner 기본 jq 유무 확인 후 조건부 설치로 최적화.
