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

| Plugin | Version | npm Package |
|--------|---------|-------------|
| hetzner | 1.2.0 | hetzner-mcp-server |
| coolify | 1.3.1 | jurislm-coolify-mcp |

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
