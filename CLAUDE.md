# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# 驗證 JSON 格式
cat .claude-plugin/marketplace.json | jq .
cat plugins/<name>/.claude-plugin/plugin.json | jq .

# 檢查版本是否同步
grep -A1 '"name": "<plugin>"' .claude-plugin/marketplace.json
cat plugins/<name>/.claude-plugin/plugin.json | grep version
```

## Repository Overview

This is a Claude Code Plugin Marketplace (`jurislm-plugins`) containing plugins that integrate MCP (Model Context Protocol) servers with Claude Code.

## Structure

```
.claude-plugin/marketplace.json  # Marketplace definition (name, owner, plugin list)
plugins/
  <plugin-name>/
    .claude-plugin/plugin.json   # Plugin metadata (name, description, version, author)
    .mcp.json                    # MCP server configuration
    README.md                    # Plugin documentation
```

## Adding a New Plugin

1. Create directory: `plugins/<plugin-name>/`
2. Add `.claude-plugin/plugin.json`:
   ```json
   {
     "name": "<plugin-name>",
     "description": "...",
     "version": "1.0.0",
     "author": { "name": "..." }
   }
   ```
3. Add `.mcp.json` with MCP server config:
   ```json
   {
     "<plugin-name>": {
       "command": "npx",
       "args": ["-y", "<npm-package>"],
       "env": { "API_TOKEN": "${API_TOKEN}" }
     }
   }
   ```
4. Register in `.claude-plugin/marketplace.json` under `plugins` array
5. **Keep version numbers in sync** between `marketplace.json` and `plugin.json`

## Version Management

When releasing updates, increment version in **both** files:
- `.claude-plugin/marketplace.json` → `plugins[].version`
- `plugins/<name>/.claude-plugin/plugin.json` → `version`

## Environment Variables

Plugins using MCP servers typically require API tokens. Use `${VAR_NAME}` syntax in `.mcp.json` to reference environment variables. Users must set these in `~/.zshenv` or `~/.zshrc` for Claude Code to access them.

Current plugin requirements:
- **hetzner**: `HETZNER_API_TOKEN` (not `HCLOUD_TOKEN`)
- **coolify**: `COOLIFY_ACCESS_TOKEN`, `COOLIFY_BASE_URL`

## Current Plugins

| Plugin | Version | Type | 說明 |
|--------|---------|------|------|
| hetzner | 1.2.0 | MCP Server | hetzner-mcp-server（14 工具） |
| coolify | 1.3.2 | MCP Server | jurislm-coolify-mcp（35 工具） |
| lawyer | 1.0.0 | Skill Only | Payload CMS + 部署 + E2E 測試指南 |
| stock | 1.0.0 | Skill Only | TWSE/Yahoo API + 投資組合 + E2E 測試 |

## Gotchas

- **版本號必須同步**：`marketplace.json` 和 `plugin.json` 的版本必須一致
- **環境變數名稱**：hetzner 用 `HETZNER_API_TOKEN`（不是 `HCLOUD_TOKEN`）
- **description 語言**：使用繁體中文

## 經驗教訓

### MCP 獨立安裝 → Plugin 遷移

從獨立 MCP Server（`.claude.json` 的 `mcpServers`）遷移到 Plugin 系統時，舊配置**不會自動移除**。

**症狀**：`/plugin` → Installed 中出現 disabled 的舊 MCP Server

**必須手動清除**：
1. `.claude.json` 中該專案的 `mcpServers` 物件（刪除整個 server 定義）
2. `.claude.json` 中該專案的 `disabledMcpServers` 陣列（移除對應條目）

**驗證**：重啟 Claude Code → `/plugin` → Installed 確認只有 Plugin 版

### 環境變數配置注意

- 環境變數定義在 `~/.zshenv`（推薦）而非 `~/.zshrc`，確保非互動式 shell 也能讀取
- Claude Code 的 MCP Server 是子進程，`~/.zshrc` 可能不會被 source

---

## 跨專案開發經驗模式（2026-02-09 更新）

> 從各專案開發中提煉的通用教訓，避免重複犯錯。

### ESLint 配置

**globalIgnores 是逃避不是解法**
- 完全 ignore 一個目錄 = 失去所有 lint 保護（formatting、type safety、best practices 全消失）
- 正確做法：為目標檔案設定專屬 ESLint override，只關閉不適用的規則
- 例：E2E 測試 → `eslint-plugin-playwright` + 關閉 `react-hooks`，保留 TS/formatting 檢查
- 來源：stock 專案 PR #12 — Copilot 指出 `tests/e2e/**` 不該全域忽略

**Vitest exclude 不可覆蓋預設**
- `exclude: ['tests/e2e/**', 'node_modules/**']` 會覆蓋 vitest 預設排除的目錄
- 正確：`import { configDefaults } from 'vitest/config'` → `exclude: [...configDefaults.exclude, 'tests/e2e/**']`

### React 狀態管理

**useSyncExternalStore snapshot 必須返回穩定引用**
- `getSnapshot` 每次返回新物件 → `Object.is` 永遠 false → 無限 re-render → `Maximum update depth exceeded`
- 解法：cache raw string + parsed result，只在 raw string 改變時才創建新物件
- 來源：stock 專案 settings page — `readSettings()` 每次 `JSON.parse` 產生新物件

### E2E 測試

**Mock 資料格式必須與 API 回傳型別一致**
- 寫 mock 前先讀 API route handler 的回傳型別（陣列 vs 物件）
- `{ taiex: {...}, otc: {...} }` vs `[{...}]` 會導致 `undefined.name` runtime crash
- 最佳實踐：從 TypeScript 型別定義反推 mock 結構

**Playwright assertion 陷阱**
- `expect(x).toBeVisible().catch(() => {})` — catch 吞掉斷言失敗，測試永遠不會 fail
- `not.toBeVisible()` → 用 `toBeHidden()` 更語義化（`playwright/no-useless-not`）
- 每個 test 至少一個 assertion（`playwright/expect-expect`）
- `toHaveClass()` 等 async matcher 必須 `await`（`playwright/missing-playwright-await`）

**Playwright route pattern**
- `page.route('/api/**')` 只攔截相對路徑 → 用 `page.route('**/api/**')` 攔截完整 URL
- 未 mock 路由應回傳 404（hermetic tests），不要 `route.fallback()`

### PR Review 流程

**修正後必須逐項讀檔驗證**
- 修完 N 項後必須逐一 `Read` 對應檔案行號確認實際變更
- 聲稱「全部修完」但漏了 1 項 = 信任崩壞
- 用表格追蹤：`| # | 檔案:行號 | 建議 | 狀態 | 驗證 |`

### Git 工作流

**Merge + Review fixes 分開 commit**
- merge commit 不應包含非 merge 的修改
- 策略：`git stash` review fixes → 完成 merge commit → `git stash pop` → 建立 review fixes commit
