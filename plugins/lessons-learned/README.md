# Lessons Learned Plugin

跨專案開發經驗模式庫，從實際踩坑中提煉的關鍵教訓與改進方案。

## 功能

- **53 個經驗模式**，按 11 個主題分類
- 每個模式包含：問題描述、根因分析、正確解法、來源專案
- 涵蓋：診斷除錯、測試策略、基礎設施、安全防護、架構重構、Git 工作流、業務邏輯、雲端遷移、前端工具鏈、Docker 部署

## 安裝

```
/plugin install lessons-learned@jurislm-plugins
```

## 適用場景

- 遇到問題時查找是否有已知模式
- PR Review 時避免重複犯錯
- 新專案設定時參考最佳實踐
- 除錯卡關時提供系統性診斷思路

## 分類

| 類別 | 模式數 | 涵蓋主題 |
|------|--------|----------|
| A. 診斷與除錯 | 8 | 環境 vs 代碼、過度工程化、系統性診斷、Typecheck 失敗 |
| B. 測試 | 12 | Vitest、Playwright、E2E mock、assertion 陷阱、globalIgnores |
| C. 基礎設施與部署 | 7 | MCP、Coolify、Hetzner、Release Please、Migration 安全 |
| D. 安全與錯誤處理 | 4 | Production 錯誤隱藏、Token 估算、SSE 解析、Regex lastIndex |
| D2. 業務邏輯 | 4 | JurisLM 核心業務、Legal Plugin 設計、分類系統、Prompt Injection |
| E. 架構與重構 | 5 | Config-driven mapping、Token budget、Per-tool timeout、API 錯誤分類 |
| F. Git 工作流 | 3 | Merge + review 分離、PR Review 驗證 |
| G. 工具與工作流 | 2 | OpenSpec、Task tool subagent_type |
| H. 雲端遷移與環境配置 | 6 | pg_class.reltuples、Bun.serve idleTimeout、Payload CMS Migration |
| I. 前端工具鏈與框架 | 3 | Tailwind CSS v4、設計系統一致性、TypeScript 錯誤診斷 |
| J. Turborepo Docker 部署 | 4 | turbo prune、Coolify ARG 注入、COPY node_modules、serveStatic 路徑 |

## 授權

MIT
