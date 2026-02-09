# Lessons Learned Plugin

跨專案開發經驗模式庫，從實際踩坑中提煉的關鍵教訓與改進方案。

## 功能

- 49+ 經驗模式，按主題分類
- 每個模式包含：問題描述、根因分析、正確解法、來源專案
- 涵蓋：診斷除錯、測試策略、基礎設施、安全防護、架構重構、Git 工作流

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
| 診斷與除錯 | 8 | 環境 vs 代碼、過度工程化、系統性診斷 |
| 測試 | 12 | Vitest、Playwright、E2E mock、assertion 陷阱 |
| 基礎設施 | 7 | MCP、Coolify、Hetzner、Release Please |
| 安全與錯誤處理 | 8 | finally block、timeout、rate limiting、sanitize、error sanitization、API 錯誤分類 |
| 架構與重構 | 5 | config-driven mapping、token budget、CJK estimation、per-tool timeout、SSE state machine |
| Git 工作流 | 2 | merge + review 分離、PR Review 驗證 |

## 授權

MIT
