# Lawyer Plugin

劉尹惠律師事務所網站（lawyer-app）開發 Skill。

## 功能

提供 lawyer-app 專案的開發指南，包含：

- **Payload CMS** — Collection 設計、Slug 生成、Admin CSS 修復、importMap 白屏陷阱、prodMigrations
- **部署工作流** — Coolify 環境設定、Staging/Production 部署、三層 Staging 保護
- **E2E 測試** — Playwright 90 項測試的架構、陷阱與最佳實踐
- **設計系統** — Sage green 色彩、LXGW WenKai TC 字體、深色模式、typography classes
- **GitHub Actions** — Release Please 自動版本管理、Claude Code Review

## 安裝

```bash
/plugin install lawyer@jurislm-plugins
```

## 技術棧

- Next.js 16.1.6 + App Router + Turbopack
- Payload CMS 3.75.0（嵌入式）
- PostgreSQL（Coolify 託管）
- Tailwind CSS 4 + SCSS
- next-themes（深色模式）
- Playwright 1.58.2（E2E Testing）
- Vitest 4.0.18（Unit Testing）

## 使用方式

安裝後，在 lawyer-app 專案中自動提供 `/lawyer` skill，涵蓋開發、部署、測試的完整知識庫。

## 版本

- **1.2.0** — 全面更新反映 codebase 現狀：技術版本號、Production DB、三層 Staging 保護、prodMigrations、GitHub Actions、設計系統更新
- **1.1.0** — 新增 Staging 三層保護文件、Cloudflare WAF 注意事項
- **1.0.0** — 初始版本，包含 Payload CMS 遷移經驗、E2E 測試架構、部署工作流
