# Release Please Workflow

## 檔案位置

`.github/workflows/release.yml`

## 用途

自動依據 Conventional Commits 產生版本號與 changelog，建立 release PR。推送到 main 時觸發。

## 完整內容

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

## 自訂指引

- **release-type**：`node` 適用有 `package.json` 的專案；其他語言改為 `python`、`go`、`rust` 等
- **Conventional Commits 規則**：
  - `feat:` → MINOR（0.1.0 → 0.2.0）
  - `fix:` → PATCH（0.1.0 → 0.1.1）
  - `feat!:` 或 `BREAKING CHANGE:` → MAJOR（0.1.0 → 1.0.0）
- **前置設定**：GitHub Repo → Settings → Actions → General → Workflow permissions → 啟用「Allow GitHub Actions to create and approve pull requests」
- **GITHUB_TOKEN**：自動提供，不需手動設定 secret

## 來源

`stock/.github/workflows/release.yml`
