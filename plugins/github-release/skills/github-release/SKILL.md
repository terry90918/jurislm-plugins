---
name: github-release
description: This skill should be used when the user asks to "set up GitHub Actions", "add release workflow", "add CI/CD", "configure Release Please", "add Claude Code review", "set up automated releases", or wants to standardize GitHub workflows for a new or existing project. Activate when user mentions GitHub Actions, release automation, or CI/CD setup.
---

# GitHub Release 標準化工作流指南

為所有專案提供一致的 GitHub Actions 配置，涵蓋自動版本管理、Claude Code 整合與 Release Notes 分類。

## 標準配置檔清單

每個專案應包含以下 4 個檔案：

| 檔案 | 觸發條件 | 用途 |
|------|----------|------|
| `.github/workflows/release.yml` | push to main | Release Please 自動建立 release PR |
| `.github/workflows/claude.yml` | @claude 留言 | Claude Code 回應 issue/PR 中的 @claude 指令 |
| `.github/workflows/claude-code-review.yml` | PR opened/synced | Claude Code 自動 PR Review |
| `.github/release.yml` | release 建立時 | GitHub Release Notes 自動分類 |

## 檔案模板

### 1. Release Please（`.github/workflows/release.yml`）

自動依據 Conventional Commits 產生版本號與 changelog，建立 release PR。

```yaml
name: Release Please

on:
  push:
    branches:
      - main
  workflow_dispatch:

permissions:
  contents: write
  pull-requests: write

jobs:
  release-please:
    runs-on: ubuntu-latest
    steps:
      - uses: googleapis/release-please-action@v4
        with:
          release-type: node
          token: ${{ secrets.GITHUB_TOKEN }}
```

**Conventional Commits 規則**：
- `feat:` → MINOR 版本（0.1.0 → 0.2.0）
- `fix:` → PATCH 版本（0.1.0 → 0.1.1）
- `feat!:` 或 `BREAKING CHANGE:` → MAJOR 版本（0.1.0 → 1.0.0）

**注意**：需在 GitHub Repo → Settings → Actions → General → Workflow permissions 啟用「Allow GitHub Actions to create and approve pull requests」。

### 2. Claude Code Action（`.github/workflows/claude.yml`）

在 issue/PR 中使用 @claude 與 Claude Code 互動。

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
      actions: read
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
          additional_permissions: |
            actions: read
```

### 3. Claude Code Review（`.github/workflows/claude-code-review.yml`）

PR 開啟或更新時自動觸發 Claude Code Review。

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

### 4. Release Notes 分類（`.github/release.yml`）

GitHub 自動產生 Release Notes 時的分類規則。

```yaml
changelog:
  exclude:
    labels:
      - ignore-for-release
      - dependencies
    authors:
      - dependabot
      - renovate

  categories:
    - title: '⚠️ Breaking Changes'
      labels:
        - breaking

    - title: '🚀 New Features'
      labels:
        - feat

    - title: '🐛 Bug Fixes'
      labels:
        - fix

    - title: '⚡ Performance'
      labels:
        - perf

    - title: '📚 Documentation'
      labels:
        - docs

    - title: '♻️ Refactoring'
      labels:
        - refactor

    - title: '🧪 Tests'
      labels:
        - test

    - title: '🔧 CI/CD'
      labels:
        - ci

    - title: '🏠 Maintenance'
      labels:
        - chore

    - title: '📦 Other Changes'
      labels:
        - '*'
```

## 建置工作流

為新專案設定時，依序執行：

1. 建立 `.github/workflows/` 目錄
2. 複製上述 4 個檔案
3. 在 GitHub Repo Settings 設定：
   - **Secrets**: 新增 `CLAUDE_CODE_OAUTH_TOKEN`
   - **Actions permissions**: 啟用「Allow GitHub Actions to create and approve pull requests」
4. 確認 `package.json` 存在（Release Please `release-type: node` 需要）

## 前置需求

### Secrets 設定

| Secret | 用途 | 取得方式 |
|--------|------|----------|
| `GITHUB_TOKEN` | Release Please | 自動提供，不需手動設定 |
| `CLAUDE_CODE_OAUTH_TOKEN` | Claude Code Action | [Claude Code OAuth](https://console.anthropic.com/) |

### Repo Settings

- **Actions → General → Workflow permissions**:
  - ✅ Read and write permissions
  - ✅ Allow GitHub Actions to create and approve pull requests

## 已套用的專案

| 專案 | 狀態 |
|------|------|
| terry90918/stock | ✅ 4 檔齊全 |
| terry90918/lawyer-app | ✅ 4 檔齊全 |

## 注意事項

- `release-type: node` 適用於有 `package.json` 的專案；其他語言請改為對應類型（如 `python`、`go` 等）
- Claude Code Review 的 `prompt` 使用了 `${{ github.repository }}` 和 `${{ github.event.pull_request.number }}`，這些是 GitHub context 變數，安全且不涉及使用者輸入注入
- `.github/release.yml` 不是 workflow，是 GitHub Release Notes 的配置檔，放在 `.github/` 根目錄下
