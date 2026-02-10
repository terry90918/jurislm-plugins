# Release Notes 分類配置

## 檔案位置

`.github/release.yml`

## 用途

GitHub 自動產生 Release Notes 時的分類規則。依據 PR labels 將變更自動歸類到對應分類。**注意**：此檔案不是 workflow，放在 `.github/` 根目錄下（非 `workflows/`）。

## 完整內容

```yaml
# GitHub Auto-generated Release Notes Configuration
# https://docs.github.com/en/repositories/releasing-projects-on-github/automatically-generated-release-notes
#
# Labels follow Semantic Versioning (https://semver.org/):
# - MAJOR: breaking changes (incompatible API changes)
# - MINOR: new features (backwards compatible)
# - PATCH: bug fixes (backwards compatible)

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

## 自訂指引

- **Labels 必須先建立**：需在 GitHub Repo → Labels 中建立對應的 label（`feat`, `fix`, `docs` 等）
- **PR 必須加 label**：每個 PR 至少加一個 label，否則歸入「Other Changes」
- **排除規則**：`dependabot` 和 `renovate` 的 PR 自動排除，避免 release notes 被依賴更新淹沒
- **與 Release Please 搭配**：Release Please 建立的 release 會自動套用此分類
- **自訂分類**：可依專案需求新增或移除分類（如移除 `perf`、新增 `security`）

## 來源

`lawyer-app/.github/release.yml`
