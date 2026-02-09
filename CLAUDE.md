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
    .mcp.json                    # MCP server configuration (MCP Server type only)
    README.md                    # Plugin documentation
    skills/                      # Skill definitions (Skill Only type only)
      <skill-name>/
        SKILL.md                 # Skill definition with YAML frontmatter
        references/              # Reference files (optional)
```

**Plugin Types**:
- **MCP Server**: `.mcp.json` + `plugin.json` + `README.md`（如 hetzner, coolify）
- **Skill Only**: `plugin.json` + `README.md` + `skills/`（如 lawyer, stock, jurislm-dev）

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
| jurislm-dev | 1.1.0 | Skill Only | Unified Agent + CLI + Dashboard + 資料同步 + 法律分類（3 skills） |
| github-release | 1.0.0 | Skill Only | Release Please + Claude Code Review + Release Notes |
| lessons-learned | 1.4.0 | Skill Only | 56 經驗模式：診斷除錯、測試、基礎設施、安全、架構、業務邏輯、雲端遷移、前端工具鏈、Docker 部署 |

## Gotchas

- **版本號必須同步**：`marketplace.json` 和 `plugin.json` 的版本必須一致
- **環境變數名稱**：hetzner 用 `HETZNER_API_TOKEN`（不是 `HCLOUD_TOKEN`）
- **description 語言**：使用繁體中文
- **Plugin 名稱不可與 marketplace 同名**：`jurislm` plugin 在 `jurislm-plugins` marketplace 內會造成歧義，對話中無法區分指的是哪個。命名時加後綴區分（如 `jurislm-dev`）
- **Skill-Only plugin 不需要 `.mcp.json`**：只有 MCP Server 類型的 plugin 需要 `.mcp.json`。Skill-Only plugin 結構為 `plugin.json` + `README.md` + `skills/` 目錄

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

### Project Skills → Plugin 遷移

從 project-scoped skills（`.claude/skills/`）遷移到 marketplace plugin 時：

**命名陷阱**
- Plugin 名稱不可與 marketplace 名稱重疊（例：`jurislm` plugin 在 `jurislm-plugins` marketplace → 歧義）
- 解法：加後綴 `-dev`、`-platform` 等區分，例 `jurislm-dev`
- 來源：jurislm plan-a 遷移 — 初次命名 `jurislm` 造成對話混淆，事後改名

**遷移內容必須更新過時引用**
- SKILL.md 中的觸發詞、技術棧表、目錄結構必須反映當前架構
- reference 檔案若描述舊架構（如 LangGraph 12-Node → Unified Agent），必須重寫而非原封複製
- 原封複製的檔案（CLI commands、database schema 等穩定內容）可直接 `cp`

**Skill-Only plugin 結構**
```
plugins/<name>/
├── .claude-plugin/plugin.json   # name, description, version, author
├── README.md                    # 安裝指令、適用場景、技術棧
└── skills/
    └── <skill-name>/
        ├── SKILL.md             # 含 YAML frontmatter（name, description, version）
        └── references/          # 參考文件（可選）
```
- 無需 `.mcp.json`（MCP Server 類型才需要）
- 一個 plugin 可包含多個 skills（如 jurislm-dev 包含 3 個）

**安裝流程**
1. `git push` 到 marketplace repo
2. `/plugin marketplace update <marketplace-name>` — 必須先更新 marketplace 索引
3. `/plugin install <plugin-name>@<marketplace-name>` — 才能安裝
4. 重啟 Claude Code — skills 才會載入
- 跳過步驟 2 會導致 `Plugin not found`

**清理 project skills**
- `git rm -r .claude/skills/<name>` 刪除舊 skills
- 保留同目錄下不相關的 skills（如 `openspec-*`、`document-generation`）
- 分開 commit：plugin 建立（jurislm-plugins repo）和 skill 刪除（app repo）應各自獨立 commit

---

## 跨專案開發經驗模式

> 更多詳細的開發經驗模式（56 個模式，11 個分類）已整合至 **lessons-learned plugin**。

包含主題：
- 診斷與除錯、測試策略、基礎設施與部署
- 安全與錯誤處理、業務邏輯、架構與重構
- Git 工作流、工具與工作流、雲端遷移與環境配置
- 前端工具鏈與框架、Turborepo Docker 部署

**使用方式**：
```bash
/plugin install lessons-learned@jurislm-plugins
```

安裝後可在任何專案中使用這些經驗模式，涵蓋從除錯、測試、部署到架構重構的完整開發生命週期。
