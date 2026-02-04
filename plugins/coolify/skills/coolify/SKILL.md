---
name: coolify
description: This skill should be used when the user asks to "deploy to Coolify", "manage Coolify applications", "check Coolify status", "create database on Coolify", "manage Coolify servers", "diagnose Coolify issues", "update environment variables on Coolify", or mentions Coolify deployment, infrastructure management, or self-hosted PaaS operations.
---

# Coolify MCP Server 使用指南

Coolify 是一個開源的自託管 PaaS（Platform as a Service），類似於 Heroku 或 Vercel。此 skill 提供透過 MCP 工具管理 Coolify 基礎設施的完整指南。

## MCP 工具概覽

@jurislm/coolify-mcp 提供 35 個最佳化工具，分為以下類別：

| 類別 | 功能 |
|------|------|
| 基礎設施 | 版本檢查、基礎設施概覽 |
| 診斷 | 應用/伺服器診斷、問題掃描 |
| 伺服器 | 列表、詳情、資源、域名、驗證 |
| 應用程式 | CRUD、日誌、環境變數、控制 |
| 資料庫 | 8 種類型、備份排程管理 |
| 服務 | 建立、更新、刪除、控制 |
| 部署 | 列表、部署、取消、狀態 |
| 批量操作 | 重啟專案、環境變數更新、全面停止 |

## 智能查詢功能

除了 UUID，MCP 支援人類易讀識別符：
- **應用域名**：`stuartmason.co.uk` 而非 UUID
- **伺服器 IP**：`192.168.1.100` 而非 UUID

## 常見工作流程

### 部署新應用程式

1. 使用 `coolify_infrastructure_overview` 查看可用伺服器和專案
2. 使用 `coolify_create_application` 建立應用程式
3. 使用 `coolify_update_env_vars` 設定環境變數
4. 使用 `coolify_deploy` 觸發部署
5. 使用 `coolify_deployment_status` 監控部署進度

### 診斷問題

1. 使用 `coolify_scan_for_issues` 掃描基礎設施問題
2. 使用 `coolify_diagnose_application` 診斷特定應用程式
3. 使用 `coolify_get_application_logs` 查看應用程式日誌
4. 使用 `coolify_server_resources` 檢查伺服器資源

### 管理資料庫

1. 使用 `coolify_list_databases` 列出所有資料庫
2. 使用 `coolify_create_database` 建立新資料庫（支援 PostgreSQL、MySQL、MongoDB、Redis 等）
3. 使用 `coolify_manage_backup_schedule` 設定備份排程

### 批量操作

- `coolify_restart_project_apps` - 重啟專案中所有應用程式
- `coolify_batch_env_update` - 批量更新環境變數
- `coolify_stop_all` - 停止所有服務（維護用）

## 環境變數設定

使用前需設定以下環境變數：

```bash
# ~/.zshenv
export COOLIFY_ACCESS_TOKEN="your-api-token"
export COOLIFY_BASE_URL="https://your-coolify-instance.com"
```

取得 API Token：Coolify Dashboard → Settings → API Tokens → Generate

## 最佳實踐

### 部署策略

- **藍綠部署**：使用兩個環境輪流部署，透過域名切換
- **滾動更新**：設定 `health_check_path` 確保平滑更新
- **回滾**：保留前一版本的部署設定

### 資源管理

- 定期使用 `coolify_server_resources` 監控資源使用
- 設定資源限制（CPU、記憶體）避免單一應用影響整體
- 使用 `coolify_scan_for_issues` 定期檢查問題

### 安全考量

- 使用環境變數儲存敏感資料，不要硬編碼
- 定期輪換 API Token
- 啟用 SSL/TLS（Let's Encrypt 自動管理）

## 故障排除

### 部署失敗

1. 檢查應用程式日誌：`coolify_get_application_logs`
2. 驗證建置設定（Dockerfile、Nixpacks）
3. 確認資源足夠：`coolify_server_resources`
4. 檢查網路連接和防火牆

### 應用程式無法存取

1. 診斷應用程式：`coolify_diagnose_application`
2. 檢查域名設定和 SSL 狀態
3. 驗證健康檢查路徑
4. 檢查容器狀態

### 資料庫連接問題

1. 確認資料庫正在運行
2. 檢查連接字串格式
3. 驗證網路政策（內部/外部存取）
4. 檢查認證資訊

## 附加資源

### 參考文件

詳細資訊請參閱：
- **`references/api-reference.md`** - 完整 API 端點文件
- **`references/deployment-patterns.md`** - 部署模式與最佳實踐
- **`references/troubleshooting.md`** - 詳細故障排除指南

### 外部連結

- [Coolify 官方文件](https://coolify.io/docs)
- [@jurislm/coolify-mcp GitHub](https://github.com/terry90918/coolify-mcp)
- [Coolify API 文件](https://coolify.io/docs/api)
