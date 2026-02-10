# GitHub Release Plugin

標準化 GitHub Actions 工作流配置 Skill，確保所有專案使用一致的 CI/CD 設定。

## 功能

為新專案或現有專案快速建立 5 個標準化配置檔：

| 檔案 | 用途 |
|------|------|
| `.husky/pre-commit` | Husky pre-commit hook（格式化、lint、typecheck、test） |
| `.github/workflows/release.yml` | Release Please 自動版本管理 |
| `.github/workflows/claude.yml` | Claude Code GitHub Action（@claude 互動） |
| `.github/workflows/claude-code-review.yml` | Claude Code 自動 PR Review |
| `.github/release.yml` | GitHub Release Notes 分類配置 |

## 安裝

```
/plugin install github-release@jurislm-plugins
```

## 使用方式

```
# 為專案建立標準化 GitHub Actions
「幫這個專案加上 GitHub release 和 CI 設定」

# 檢查現有配置是否完整
「檢查這個 repo 的 GitHub workflows 是否齊全」
```

## 前置需求

| Secret | 用途 | 設定位置 |
|--------|------|----------|
| `GITHUB_TOKEN` | Release Please（自動提供） | 不需手動設定 |
| `CLAUDE_CODE_OAUTH_TOKEN` | Claude Code Action | GitHub Repo → Settings → Secrets |

## 授權

MIT
