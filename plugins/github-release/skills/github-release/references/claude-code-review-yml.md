# Claude Code Review Workflow

## 檔案位置

`.github/workflows/claude-code-review.yml`

## 用途

PR 開啟或更新時自動觸發 Claude Code Review，使用官方 `code-review` plugin 進行自動化代碼審查。

## 完整內容

```yaml
name: Claude Code Review

on:
  pull_request:
    types: [opened, synchronize, ready_for_review, reopened]

jobs:
  claude-review:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: read
      issues: read
      id-token: write

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 1

      - name: Run Claude Code Review
        id: claude-review
        uses: anthropics/claude-code-action@v1
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          plugin_marketplaces: 'https://github.com/anthropics/claude-code.git'
          plugins: 'code-review@claude-code-plugins'
          prompt: '/code-review:code-review ${{ github.repository }}/pull/${{ github.event.pull_request.number }}'
```

## 自訂指引

- **路徑篩選**：取消註解 `paths:` 可限制只在特定檔案變更時觸發 review（如 `src/**/*.ts`）
- **作者篩選**：取消註解 `if:` 可限制只對特定作者的 PR 觸發（如外部貢獻者）
- **plugin_marketplaces**：指向 Anthropic 官方 claude-code plugin marketplace
- **plugins**：使用 `code-review@claude-code-plugins` 官方 code review plugin
- **CLAUDE_CODE_OAUTH_TOKEN**：與 claude.yml 共用同一個 secret

## 來源

`stock/.github/workflows/claude-code-review.yml`
