# Stock Plugin

台股看盤應用（stock.jurislm.com）的 Claude Code 開發 Skill。

## 功能

- 完整架構指南（API → Hooks → Components 資料流）
- 目前約 69 個 API routes 的開發慣例與除錯重點
- 目前 20 個 hooks 的狀態管理與 TTL 快取模式
- Prisma 資料庫 schema 與關聯
- E2E 測試（Playwright）與單元測試（Vitest）策略
- 常見開發工作流與陷阱

## 安裝

```
/plugin install stock@jurislm-plugins
```

## 適用場景

- 開發台股看盤相關功能（報價、投資組合、觀察清單、ETF 分析）
- 新增 API endpoint 或 React hook
- 撰寫或除錯 E2E / 單元測試
- 理解 TWSE/Yahoo Finance API 整合模式

## 技術棧

| 類別 | 技術 |
|------|------|
| 框架 | Next.js 16.1.6 (App Router) + React 19.2.3 |
| 資料庫 | PostgreSQL + Prisma ORM (`@prisma/client` 6.19.2) |
| 測試 | Vitest 4.0.16 + RTL / Playwright 1.58.2（14 個 E2E spec files） |
| 圖表 | lightweight-charts / d3-force |
| 套件管理 | Bun |

## 相關連結

- [Stock App](https://stock.jurislm.com)
- [GitHub Repo](https://github.com/terry90918/stock)

## 授權

MIT
