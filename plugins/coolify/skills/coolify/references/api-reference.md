# Coolify MCP API 參考

## 基礎設施工具

### coolify_version
檢查 Coolify 實例版本。

### coolify_infrastructure_overview
取得基礎設施概覽，包括：
- 所有伺服器及其狀態
- 所有專案及應用程式數量
- 資源使用摘要

## 伺服器管理

### coolify_list_servers
列出所有伺服器。

回傳欄位：
- `id` - 伺服器 UUID
- `name` - 伺服器名稱
- `ip` - IP 地址
- `status` - 狀態（running/stopped）

### coolify_server_details
取得特定伺服器詳情。

參數：
- `server_id` - 伺服器 UUID 或 IP

### coolify_server_resources
取得伺服器資源使用情況。

回傳：
- CPU 使用率
- 記憶體使用率
- 磁碟使用率
- 網路流量

### coolify_server_domains
列出伺服器上所有域名。

### coolify_validate_server
驗證伺服器連接和設定。

## 應用程式管理

### coolify_list_applications
列出所有應用程式。

可選參數：
- `project_id` - 過濾特定專案
- `server_id` - 過濾特定伺服器

### coolify_get_application
取得應用程式詳情。

參數：
- `app_id` - 應用程式 UUID 或域名

### coolify_create_application
建立新應用程式。

參數：
- `name` - 應用程式名稱
- `project_id` - 專案 UUID
- `server_id` - 伺服器 UUID
- `git_repository` - Git 倉庫 URL
- `git_branch` - Git 分支（預設：main）
- `build_pack` - 建置類型（nixpacks/dockerfile/static）
- `domains` - 域名列表

### coolify_update_application
更新應用程式設定。

### coolify_delete_application
刪除應用程式。

### coolify_start_application
啟動應用程式。

### coolify_stop_application
停止應用程式。

### coolify_restart_application
重啟應用程式。

### coolify_get_application_logs
取得應用程式日誌。

參數：
- `app_id` - 應用程式 UUID 或域名
- `lines` - 日誌行數（預設：100）
- `since` - 起始時間

## 環境變數

### coolify_list_env_vars
列出應用程式環境變數。

參數：
- `app_id` - 應用程式 UUID 或域名

### coolify_update_env_vars
更新環境變數。

參數：
- `app_id` - 應用程式 UUID 或域名
- `env_vars` - 環境變數物件 `{"KEY": "value"}`

### coolify_batch_env_update
批量更新多個應用程式的環境變數。

## 部署

### coolify_list_deployments
列出部署歷史。

參數：
- `app_id` - 應用程式 UUID 或域名
- `limit` - 數量限制

### coolify_deploy
觸發部署。

參數：
- `app_id` - 應用程式 UUID 或域名
- `force` - 強制重新建置（預設：false）

### coolify_deployment_status
取得部署狀態。

參數：
- `deployment_id` - 部署 UUID

### coolify_cancel_deployment
取消進行中的部署。

## 資料庫管理

### coolify_list_databases
列出所有資料庫。

支援類型：
- PostgreSQL
- MySQL / MariaDB
- MongoDB
- Redis
- Dragonfly
- KeyDB
- ClickHouse
- Valkey

### coolify_get_database
取得資料庫詳情。

### coolify_create_database
建立新資料庫。

參數：
- `name` - 資料庫名稱
- `type` - 資料庫類型
- `project_id` - 專案 UUID
- `server_id` - 伺服器 UUID
- `version` - 版本（可選）

### coolify_start_database / coolify_stop_database / coolify_restart_database
控制資料庫服務。

### coolify_manage_backup_schedule
管理資料庫備份排程。

參數：
- `database_id` - 資料庫 UUID
- `enabled` - 啟用/停用
- `frequency` - 頻率（daily/weekly/monthly）
- `retention_days` - 保留天數

## 服務管理

### coolify_list_services
列出所有服務。

### coolify_create_service
建立新服務（從模板）。

### coolify_update_service
更新服務設定。

**支援的欄位**：
- `name` - 服務名稱
- `description` - 服務描述
- `docker_compose_raw` - Docker Compose 配置

⚠️ **重要限制**：
- **不支援直接更新 FQDN/Domain**
- Service FQDN 由 `docker_compose_raw` 中的 Traefik labels 控制
- 要更新 FQDN，需修改 `docker_compose_raw` 中的相關 labels 後重新部署
- 或透過 Coolify Web UI 直接修改 Domain 設定

### coolify_delete_service
刪除服務。

### coolify_start_service / coolify_stop_service / coolify_restart_service
控制服務。

## 診斷工具

### coolify_diagnose_application
診斷應用程式問題。

回傳：
- 健康狀態
- 最近錯誤
- 資源使用
- 建議修復

### coolify_diagnose_server
診斷伺服器問題。

### coolify_scan_for_issues
掃描整個基礎設施的問題。

回傳：
- 停止的應用程式
- 資源警告
- SSL 證書過期
- 建置失敗

## 批量操作

### coolify_restart_project_apps
重啟專案中所有應用程式。

參數：
- `project_id` - 專案 UUID

### coolify_stop_all
停止所有服務（維護模式）。

⚠️ 注意：此操作會停止所有應用程式和資料庫。

## 回應格式

所有工具回傳 JSON 格式，包含：
- `success` - 操作是否成功
- `data` - 回傳資料
- `error` - 錯誤訊息（如果有）

## 錯誤代碼

| 代碼 | 說明 |
|------|------|
| 400 | 請求參數錯誤 |
| 401 | 認證失敗（Token 無效） |
| 403 | 權限不足 |
| 404 | 資源不存在 |
| 500 | 伺服器內部錯誤 |
