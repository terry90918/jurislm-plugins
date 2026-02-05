# Coolify Plugin

透過 Claude Code 管理 Coolify 自託管 PaaS 的 MCP 伺服器。

## 功能

- 35 個最佳化工具，比原始 API 少用 85% tokens
- 智能查詢（支援域名和 IP，不只是 UUID）
- 完整基礎設施管理
- 診斷和問題掃描
- 批量操作支援

### 工具類別

| 類別 | 功能 |
|------|------|
| 基礎設施 | 版本檢查、概覽 |
| 伺服器 | 列表、詳情、資源、驗證 |
| 應用程式 | CRUD、日誌、環境變數、控制 |
| 資料庫 | 8 種類型、備份管理 |
| 服務 | 建立、更新、控制 |
| 部署 | 部署、狀態、取消 |
| 診斷 | 問題掃描、應用/伺服器診斷 |
| 批量 | 重啟專案、批量更新 |

## 安裝

```
/plugin install coolify@jurislm-plugins
```

## 設定

1. 從 Coolify Dashboard → Settings → API Tokens 取得 API Token

2. 新增至 `~/.zshenv`：
   ```bash
   export COOLIFY_ACCESS_TOKEN="your-api-token"
   export COOLIFY_BASE_URL="https://your-coolify-instance.com"
   ```

3. 重新啟動 Claude Code

## 使用範例

```
# 查看基礎設施概覽
「給我基礎設施概覽」

# 部署應用程式
「部署 myapp.example.com 應用」

# 診斷問題
「診斷 myapp.example.com 的問題」

# 更新環境變數
「更新 myapp 的 DATABASE_URL」

# 掃描問題
「掃描基礎設施中的問題」
```

## 環境變數

| 變數 | 必需 | 說明 |
|------|------|------|
| `COOLIFY_ACCESS_TOKEN` | ✓ | API 認證令牌 |
| `COOLIFY_BASE_URL` | ✓ | Coolify 實例 URL |

## 相關連結

- [Coolify 官方文件](https://coolify.io/docs)
- [@jurislm/coolify-mcp](https://github.com/terry90918/coolify-mcp) (forked with bug fixes)
- [Coolify API 文件](https://coolify.io/docs/api)

## 授權

MIT
