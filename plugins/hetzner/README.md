# Hetzner Cloud Plugin

透過 Claude Code 管理 Hetzner Cloud 基礎設施的 MCP 伺服器。

## 功能

### 伺服器管理（7 個工具）
- 列出、查詢、建立、刪除伺服器
- 開機、關機、重新啟動

### SSH 金鑰管理（4 個工具）
- 列出、查詢、建立、刪除 SSH 金鑰

### 參考資料（3 個工具）
- 查看伺服器類型（規格與價格）
- 查看可用映像檔（作業系統）
- 查看資料中心位置

## 限制

此 MCP 伺服器**不支援**：
- Volumes（磁碟區）管理
- Firewalls（防火牆）管理
- Projects（專案）管理
- 帳單相關操作
- 進階網路功能

## 安裝

```
/plugin install hetzner@jurislm-plugins
```

## 設定

1. 從 [Hetzner Cloud Console](https://console.hetzner.cloud/) → 專案 → Security → API Tokens 取得 API Token

2. 新增至 `~/.zshenv`：
   ```bash
   export HETZNER_API_TOKEN="your_token_here"
   ```

3. 重新啟動 Claude Code

## 環境變數

| 變數 | 必需 | 說明 |
|------|------|------|
| `HETZNER_API_TOKEN` | ✓ | Hetzner Cloud API 認證令牌 |

## 相關連結

- [hetzner-mcp-server](https://github.com/nityeshaga/hetzner-mcp-server)（npm 套件）
- [Hetzner Cloud Console](https://console.hetzner.cloud/)
- [Hetzner Cloud API 文件](https://docs.hetzner.cloud/)

## 授權

MIT
