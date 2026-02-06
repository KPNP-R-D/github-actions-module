# GitHub Actions Module Docs

이 레포는 여러 레포지토리에서 재사용하는 GitHub Actions 모듈(Reusable Workflow / Composite Action)을 제공합니다.

## Workflows

- [Release · Start (Reusable)](./docs/workflows/release-start.md)
  - 릴리즈 manifest 생성 → 커밋 로그 Draft 생성 → Gemini 정제 → Notion 페이지 생성 → manifest 커밋/푸시
- [Release · Report (Reusable)](./docs/workflows/release-report.md)
  - 선택 서비스들의 배포 결과(DEPLOYED/SKIPPED/FAILED)를 manifest에 일괄 반영 → 커밋/푸시
- [Release · Finalize (Reusable)](./docs/workflows/release-finalize.md)
  - Gate(모든 서비스 DEPLOYED/SKIPPED) → status FINALIZED → bundle 태그 생성/검증 → Notion 상태 배포완료 → GitHub Release 생성
- [Release · Generate Notes & Tag (Reusable)](./docs/workflows/release-generate-notes-and-tag.md)
  - 서비스 선택 + current_tags 기반으로 커밋 로그 Draft 생성 → Gemini 정제 → Notion 페이지 생성 → bundle 태그 생성/검증 → GitHub Release 생성

## Actions

- [Gemini Generate](./docs/actions/gemini-generate.md)
  - Gemini generateContent 호출 → `outputs.text` 반환

## Versioning

- 워크플로/액션 호출은 태그 기반 권장:
  - 안정: `@v1.2.3`
  - 메이저 트래킹: `@v1` (v1 최신 패치로 이동시키는 운영이면 가능)
