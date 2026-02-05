# JurisLM Plugins Official

Claude Code 外掛市集，提供雲端基礎設施管理工具。

## 安裝

新增市集：

```
/plugin marketplace add terry90918/jurislm-plugins
```

安裝外掛：

```
/plugin install hetzner@jurislm-plugins-official
```

## 可用外掛

### hetzner

透過 Claude Code 管理 Hetzner Cloud 基礎設施的 MCP 伺服器。

**功能：**
- 列出、建立和管理伺服器
- 建立、掛載、卸載和調整磁碟區大小
- 管理防火牆規則
- 建立和管理 SSH 金鑰
- 查看可用映像檔、伺服器類型和資料中心位置
- 開機、關機和重新啟動伺服器

**設定步驟：**

1. 從 [Hetzner Cloud Console](https://console.hetzner.cloud/) → 專案 → Security → API Tokens 取得 API Token

2. 新增至 `~/.zshenv`：
   ```bash
   export HETZNER_API_TOKEN="your_token_here"
   ```

3. 重新啟動 Claude Code

## 授權

MIT
