# Claude Code Action Workflow

## 檔案位置

`.github/workflows/claude.yml`

## 用途

在 issue 或 PR 中使用 `@claude` 與 Claude Code 互動。支援 issue 留言、PR review 留言、PR review 提交、issue 建立/指派。

## 完整內容

```yaml
name: Claude Code

on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
  issues:
    types: [opened, assigned]
  pull_request_review:
    types: [submitted]

jobs:
  claude:
    if: |
      (github.event_name == 'issue_comment' && contains(github.event.comment.body, '@claude')) ||
      (github.event_name == 'pull_request_review_comment' && contains(github.event.comment.body, '@claude')) ||
      (github.event_name == 'pull_request_review' && contains(github.event.review.body, '@claude')) ||
      (github.event_name == 'issues' && (contains(github.event.issue.body, '@claude') || contains(github.event.issue.title, '@claude')))
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: read
      issues: read
      id-token: write
      actions: read # Required for Claude to read CI results on PRs
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 1

      - name: Run Claude Code
        id: claude
        uses: anthropics/claude-code-action@v1
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}

          # This is an optional setting that allows Claude to read CI results on PRs
          additional_permissions: |
            actions: read
```

## 自訂指引

- **CLAUDE_CODE_OAUTH_TOKEN**：需在 GitHub Repo → Settings → Secrets and variables → Actions 中手動設定
- **additional_permissions**：`actions: read` 讓 Claude 讀取 CI 結果，建議保留
- **自訂 prompt**：取消註解 `prompt:` 可指定預設行為（如自動更新 PR 描述）
- **claude_args**：取消註解可限制 Claude 可使用的工具（如 `--allowed-tools Bash(gh pr:*)`）
- **觸發條件**：`if` 條件確保只在含 `@claude` 的留言時觸發，避免不必要的執行

## 來源

`stock/.github/workflows/claude.yml`
