# Lawyer Plugin

劉尹惠律師事務所網站（lawyer-app）開發 Skill。

## 功能

提供 lawyer-app 專案的開發指南，包含：

- **Payload CMS** — Collection 設計、Slug 生成、Admin CSS 修復、Migration
- **部署工作流** — Coolify 環境設定、Staging/Production 部署
- **E2E 測試** — Playwright 90 項測試的架構、陷阱與最佳實踐
- **內容遷移** — Ghost CMS → Payload CMS 遷移流程

## 安裝

```bash
/plugin install lawyer@jurislm-plugins
```

## 技術棧

- Next.js 16 + App Router
- Payload CMS 3.x（嵌入式）
- PostgreSQL（Coolify 託管）
- Tailwind CSS + next-themes
- Playwright E2E Testing

## 使用方式

安裝後，在 lawyer-app 專案中自動提供 `/lawyer` skill，涵蓋開發、部署、測試的完整知識庫。

## 版本

- **1.0.0** — 初始版本，包含 Payload CMS 遷移經驗、E2E 測試架構、部署工作流
