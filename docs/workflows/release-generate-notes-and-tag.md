# Release · Generate Notes & Tag (Reusable)

선택한 서비스 기준으로 커밋 로그 Draft를 만들고, Gemini로 릴리즈 노트를 정제한 뒤 Notion 페이지를 생성하고 bundle 태그 및 GitHub Release를 생성합니다.

## What it does

- 릴리즈 키(`release_key`)를 확정합니다. (미입력 시 자동 생성)
- 서비스 카탈로그 + 체크박스 입력(`inputs_json`)을 기반으로 포함 서비스를 확정합니다.
- `current_tags`(서비스별 tag 또는 git rev)를 기준으로 커밋 로그 Draft를 생성합니다.
  - RN-BOT 관련 커밋은 제외합니다.
  - Shared 경로는 정책에 따라 별도 섹션으로 분리됩니다.
- Draft + 템플릿을 조합해 Gemini 입력을 만들고, 릴리즈 노트를 정제합니다.
- Notion DB에 페이지를 생성합니다.
  - 상단: 정제된 릴리즈 노트
  - 하단: Raw Draft(Data 1) 토글로 Draft 원문을 첨부
- `bundle/<release_key>` 태그를 생성/검증 후 push 합니다.
- GitHub Release를 생성합니다. (`--generate-notes`)

## Usage

```yaml
jobs:
  release:
    uses: 5n-project/github-actions-module/.github/workflows/release-generate-notes-and-tag.reusable.yml@v1
    with:
      catalog_path: .github/release/service-catalog.json
      prompts_dir: prompts
      manifest_dir: releases

      inputs_json: ${{ toJson(inputs) }}
      current_tags: ${{ inputs.current_tags }}
      release_key: ${{ inputs.release_key }}
      note: ${{ inputs.note }}

      notion_category: ${{ inputs.notion_category }}

    secrets:
      GEMINI_API_KEY: ${{ secrets.GEMINI_API_KEY }}
      NOTION_TOKEN: ${{ secrets.NOTION_TOKEN }}
      NOTION_DB_ID: ${{ secrets.NOTION_DB_ID }}
      NOTION_DS_NAME: ${{ secrets.NOTION_DS_NAME }}
````

## Notes

* `contents: write` 권한이 필요합니다. (태그 push / 릴리즈 생성)
* `current_tags`는 선택된 서비스에 대해 반드시 지정되어야 하며, 값은 **기존 태그** 또는 **유효한 git revision**이어야 합니다.
* Notion 페이지 생성 시 DB의 data_source_id를 resolve하여 사용합니다. (`NOTION_DS_NAME` 지정 시 해당 이름 우선)
