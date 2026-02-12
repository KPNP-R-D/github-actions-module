# ECS Deploy (Reusable)

빌드된 ECS task definition(JSON)을 **등록(register-task-definition)** 하고, 지정한 ECS Service에 **새 taskdef로 강제 배포**한 뒤 **stability까지 대기**한다. taskdef는 caller에서 만든 artifact로 전달받는 구조다.

---

## Contract

### Required permissions

Reusable workflow 내부에서 **OIDC AssumeRole**을 쓰므로 caller는 다음이 필요하다.

* `permissions: id-token: write` (필수)
* (checkout은 안 하지만, 보통 파이프라인 표준으로) `contents: read` 권장

> 이 워크플로 YAML에 permissions 선언이 없어서, **caller의 permissions가 그대로 적용**된다.

### Required artifact

Caller에서 아래 artifact가 업로드되어 있어야 한다.

* `taskdef_artifact`: `actions/upload-artifact@v4`로 업로드된 taskdef 파일 묶음
* `taskdef_path`: 다운로드 이후 워크스페이스에서의 파일 경로 (예: `taskdef.json`)

### Prerequisites

* AssumeRole 대상 IAM policy에 최소 권한 필요:

  * `ecs:RegisterTaskDefinition`
  * `ecs:UpdateService`
  * `ecs:DescribeServices` (waiter가 내부적으로 조회)
  * (taskdef에 따라) `iam:PassRole` (task execution role / task role 지정 시 필요)

---

## workflow_call inputs

### Required

| Input              | Type   | Description                        |
| ------------------ | ------ | ---------------------------------- |
| `aws_region`       | string | AWS 리전                             |
| `ecs_cluster`      | string | ECS Cluster name/arn               |
| `ecs_service`      | string | ECS Service name                   |
| `taskdef_artifact` | string | 다운로드할 artifact 이름                  |
| `taskdef_path`     | string | artifact 다운로드 후 taskdef JSON 파일 경로 |
| `environment_name` | string | GitHub Environment 이름(job에 연결)     |

---

## workflow_call outputs

| Output        | Description                |
| ------------- | -------------------------- |
| `taskdef_arn` | 새로 등록된 task definition ARN |

---

## workflow_call secrets

Required:

* `AWS_ROLE_ARN`

  * `aws-actions/configure-aws-credentials@v4`의 `role-to-assume`로 사용

---

## What it does

### Outputs / Side effects

* Artifact에서 taskdef JSON 다운로드
* ECS에 task definition 등록 → 새로운 `taskdef_arn` 생성
* ECS Service 업데이트 (`--force-new-deployment`)
* `aws ecs wait services-stable`로 배포 안정화 대기
* workflow output으로 `taskdef_arn` 제공

---

<details>
<summary><strong>Steps detail</strong></summary>

### 0) Download taskdef artifact

* `actions/download-artifact@v4`
* `name: ${{ inputs.taskdef_artifact }}`
* `path: .` (workspace 루트로 풀림)

### 1) Configure AWS credentials (OIDC)

* `aws-actions/configure-aws-credentials@v4`
* `role-to-assume: ${{ secrets.AWS_ROLE_ARN }}`
* `aws-region: ${{ inputs.aws_region }}`

### 2) Login to ECR

* `aws-actions/amazon-ecr-login@v2`
* 이 워크플로 자체는 이미지를 push/pull 하지 않지만,

  * taskdef가 새 이미지 태그를 가리키는 경우 런타임에서 ECR pull이 필요하고
  * 파이프라인 표준 단계로 유지하는 케이스가 많음

### 3) Register task definition

* `aws ecs register-task-definition --cli-input-json file://<TASKDEF_PATH>`
* 반환된 `taskDefinitionArn`을 `steps.register.outputs.taskdef_arn`으로 export
* `TASKDEF_PATH` 파일이 없으면 즉시 실패

### 4) Update ECS service

* `aws ecs update-service --task-definition <TD_ARN> --force-new-deployment`
* 결과는 `/dev/null`로 버림(로그 노이즈 감소)

### 5) Wait for stability

* `aws ecs wait services-stable`
* 서비스가 안정화될 때까지 블로킹

</details>

---

## Caller example

```yaml
jobs:
  deploy_ecs:
    needs: [ render_taskdef ]  # taskdef artifact 만든 job
    permissions:
      id-token: write
      contents: read
    uses: 5n-project/github-actions-module/.github/workflows/ecs-deploy.reusable.yml@v1.0.0
    with:
      aws_region: ap-northeast-2
      ecs_cluster: my-prod-cluster
      ecs_service: kpnp-admin
      taskdef_artifact: ${{ needs.render_taskdef.outputs.taskdef_artifact }}
      taskdef_path: taskdef.json
      environment_name: kpnp-prod
    secrets:
      AWS_ROLE_ARN: ${{ secrets.AWS_ROLE_ARN }}
```

---

## Troubleshooting

### `Taskdef not found: <path>`

* `download-artifact`로 내려받은 위치와 `taskdef_path`가 불일치.
* 체크:

  * 업로드 시 artifact에 실제 파일이 포함됐는지
  * 업로드/다운로드에서 `name`이 일치하는지
  * 다운로드 `path: .` 기준으로 상대경로가 맞는지 (예: `./taskdef.json`)

### `AccessDeniedException` / `is not authorized to perform: ecs:RegisterTaskDefinition`

* AssumeRole 대상 IAM policy에 권한 부족.
* taskdef에 `executionRoleArn` / `taskRoleArn`이 포함되어 있으면 `iam:PassRole`도 필요할 수 있음.

### `Service is not stable`로 오래 걸리거나 타임아웃

* 보통 아래 중 하나:

  * 새 task가 health check 통과 못함
  * capacity 부족(ENI/CPU/Mem/DesiredCount)
  * 이미지 pull 실패(ECR 권한/리전/태그)
* 진단:

  * `aws ecs describe-services`, `aws ecs describe-tasks`
  * CloudWatch Logs / ALB Target health / ECS events 확인

### `Cluster/Service not found`

* `ecs_cluster` / `ecs_service` 값이 이름인지 ARN인지 혼재된 케이스.
* 한쪽으로 통일(일반적으로 name 사용)하고, 리전/계정도 일치하는지 확인.

---

다음 모듈 계속 붙여넣어.
