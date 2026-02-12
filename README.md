# GitHub Actions Module Docs

이 레포는 여러 레포지토리에서 재사용하는 GitHub Actions 모듈(Reusable Workflow / Composite Action)을 제공합니다.
배포 파이프라인을 **표준화(빌드 → 태스크정의 렌더 → ECS 배포 → 태그/릴리스/알림)** 하는 것을 목표로 합니다.

---

## Workflows (Reusable)

### Deploy

* **Build & Push to ECR (Reusable)**

  * Dockerfile 또는 docker-compose(프로필별)를 감지해 멀티플랫폼 buildx로 빌드 후 ECR push
  * output: `image_ref`

* **ECS Deploy (Reusable)**

  * taskdef artifact 다운로드 → ECS taskdef 등록 → ECS service 업데이트 → 안정화 대기
  * output: `taskdef_arn`

* **Reusable Notify**

  * `push | pr | merge | deploy` 이벤트별 메시지 포맷 통일 + 텔레그램 채널 라우팅 전송

### Release Automation

* **Release · Generate Notes & Tag (Reusable)**

  * 서비스 선택 + current_tags 기반 Draft 생성 → Gemini 정제 → Notion 페이지 생성 → `bundle/<releaseKey>` 태그 생성/검증 → GitHub Release 생성

---

## Actions (Composite)

### Release / Tagging

* **resolve-release-version**

  * `{project}/vX.Y.Z` 규칙 기반으로 다음 버전 계산 (override/bump 지원)
  * outputs: `last_tag`, `next_tag`, `major_tag`, `version`

* **Publish Tags And Release**

  * `{project}/vX.Y.Z` annotated tag 생성/푸시
  * `{project}/prod` 태그를 해당 릴리스 태그로 이동(기본 force)
  * 옵션: GitHub Release 생성
  * outputs: `next_tag`, `prod_tag`, `release_created`

### ECS / Deploy Helpers

* **Render ECS Task Definition**

  * taskdef template JSON을 `envsubst`로 렌더링
  * `.env`를 `ENVIRONMENT_JSON`으로 변환(템플릿에서 `${ENVIRONMENT_JSON}`로 주입)
  * 미치환 placeholder 검사 + sanitized preview 출력
  * output: `taskdef_path`

* **Resolve ECR Image Digest**

  * ECR에서 `{repository}:{tag}`의 digest 조회
  * outputs: `image_digest`, `image_with_digest` (`repo@sha256:...`)
  * 주의: caller에서 AWS credentials(OIDC) 선행 필요

### Notifications

* **Notify Telegram**

  * Telegram Bot API `sendMessage` 호출로 메시지 전송

### LLM

* **Gemini Generate**

  * Gemini generateContent 호출 → `outputs.text` 반환

---

## Versioning

워크플로/액션 호출은 태그 기반 사용 권장:

* 안정 고정: `@v1.2.3`
* 메이저 트래킹: `@v1`
  (운영 정책상 `v1`을 최신 패치로 이동시키는 경우에만 권장)

---

## Notes / Known Issues (현 상태 기준)

* **Reusable Notify**

  * `inputs.image`를 참조하지만 inputs에 정의가 없음
  * deploy 메시지에서 `Image:` 출력 라인이 중복됨
    → 문서/구현 정리 시 같이 수정 권장

* **Resolve ECR Image Digest**

  * `jq` 설치하지만 실제로 미사용(속도/단순화 목적이면 제거 가능)

---

원하면, 지금 README 구조에 맞춰서 각 항목을 실제 문서 파일 경로(예: `./docs/actions/...`, `./docs/workflows/...`)로 연결하는 링크 버전도 같이 만들어줄게.
