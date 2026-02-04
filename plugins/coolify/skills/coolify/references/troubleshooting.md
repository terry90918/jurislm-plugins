# Coolify 故障排除指南

## 部署問題

### 建置失敗

**症狀**：部署顯示「Build Failed」

**診斷步驟**：
1. 查看建置日誌：`coolify_get_application_logs`
2. 檢查 Dockerfile 或 Nixpacks 設定
3. 驗證依賴版本相容性

**常見原因**：

#### Node.js 記憶體不足
```
FATAL ERROR: CALL_AND_RETRY_LAST Allocation failed - JavaScript heap out of memory
```
解決：增加 Node.js 記憶體限制
```bash
NODE_OPTIONS="--max-old-space-size=4096"
```

#### Python 依賴衝突
```
ERROR: Cannot install package due to conflicting dependencies
```
解決：固定依賴版本，使用 `requirements.txt` 或 `poetry.lock`

#### Docker 建置快取問題
解決：強制重新建置
```
coolify_deploy with force=true
```

### 部署卡住

**症狀**：部署狀態長時間顯示「Deploying」

**診斷步驟**：
1. 檢查伺服器資源：`coolify_server_resources`
2. 查看容器狀態
3. 檢查網路連接

**解決方案**：
- 取消部署：`coolify_cancel_deployment`
- 增加部署超時時間
- 檢查 Docker daemon 狀態

### 健康檢查失敗

**症狀**：部署完成但應用無法存取

**診斷步驟**：
1. 診斷應用程式：`coolify_diagnose_application`
2. 檢查健康檢查路徑是否正確
3. 驗證應用程式啟動時間

**解決方案**：
- 調整健康檢查間隔和超時
- 確保健康檢查端點回傳 200
- 增加啟動延遲時間

## Service 配置問題

### Service FQDN 無法更新

**症狀**：使用 `coolify_service update` 嘗試更新 FQDN 時失敗或無效

**根本原因**：
- Coolify API 的 `UpdateServiceRequest` **不支援直接更新 FQDN**
- Service FQDN 由 `docker_compose_raw` 中的 Traefik labels 控制
- 僅支援更新：`name`, `description`, `docker_compose_raw`

**解決方案**：

1. **透過 Coolify Web UI**（推薦）
   - 進入 Service 設定頁面
   - 直接修改 Domain 欄位
   - 重新部署 Service

2. **透過 API 修改 docker_compose_raw**
   ```
   1. 取得 Service 詳情：coolify_get_service
   2. 找到 docker_compose_raw 中的 Traefik labels
   3. 修改相關 labels：
      - traefik.http.routers.*.rule=Host(`new-domain.com`)
      - caddy_0=https://new-domain.com
   4. 使用 coolify_service update 更新 docker_compose_raw
   5. 重啟 Service
   ```

3. **刪除重建**（最簡單）
   - 如果配置複雜，考慮刪除 Service 後重新建立
   - 建立時使用正確的 FQDN

### Service 內 Application 與獨立 Application 混淆

**症狀**：使用 `coolify_application update` 找不到 Application

**原因**：
- Service 內的 containers（如 Ghost、MySQL）**不是獨立 Application**
- 它們是 Service 的子組件，沒有獨立的 Application UUID
- `coolify_list_applications` 不會列出 Service 內的 containers

**診斷**：
```
# 檢查資源類型
coolify_infrastructure_overview

# 如果是 Service，使用 Service 相關工具
coolify_get_service <uuid>
coolify_list_services
```

**正確做法**：
- 使用 `coolify_service` 系列工具管理 Service
- 使用 `coolify_application` 系列工具管理獨立 Application

## 網路問題

### 無法存取應用程式

**症狀**：域名無法解析或連接超時

**診斷步驟**：
1. 檢查 DNS 設定：`nslookup domain.com`
2. 驗證 SSL 證書狀態
3. 檢查 Traefik/Caddy 設定

**常見原因**：

#### DNS 未正確設定
```
A 記錄應指向 Coolify 伺服器 IP
CNAME 記錄應指向正確的域名
```

#### SSL 證書未簽發
- 等待 Let's Encrypt 簽發（最多 5 分鐘）
- 檢查 DNS 是否已傳播
- 驗證 80/443 端口開放

#### 防火牆阻擋
```bash
# 檢查防火牆規則
ufw status
iptables -L
```

### 容器間無法通訊

**症狀**：應用程式無法連接資料庫

**診斷步驟**：
1. 確認容器在同一網路
2. 檢查連接字串
3. 驗證服務名稱解析

**解決方案**：
- 使用服務名稱而非 IP
- 確保資料庫已啟動
- 檢查防火牆規則

### 502 Bad Gateway

**症狀**：訪問時顯示 502 錯誤

**診斷步驟**：
1. 檢查應用程式是否運行
2. 驗證端口設定
3. 查看 Traefik 日誌

**解決方案**：
- 確保應用監聽正確端口
- 檢查環境變數 `PORT`
- 重啟反向代理

### 503 Service Unavailable

**症狀**：訪問時顯示 503 錯誤

**原因**：
- 應用程式未啟動
- 健康檢查失敗
- 資源不足

**解決**：
- 重啟應用程式
- 檢查日誌找出錯誤
- 增加資源配額

## 資料庫問題

### 連接失敗

**症狀**：應用程式無法連接資料庫

**診斷步驟**：
1. 確認資料庫運行中
2. 檢查連接字串格式
3. 驗證認證資訊

**連接字串格式**：
```
# PostgreSQL
postgresql://user:password@hostname:5432/database

# MySQL
mysql://user:password@hostname:3306/database

# MongoDB
mongodb://user:password@hostname:27017/database

# Redis
redis://hostname:6379
```

### 資料庫效能問題

**症狀**：查詢緩慢或超時

**診斷步驟**：
1. 檢查資料庫資源使用
2. 分析慢查詢
3. 檢查索引

**解決方案**：
- 增加資料庫資源
- 優化查詢
- 新增適當索引
- 啟用查詢快取

### 備份失敗

**症狀**：自動備份未執行

**診斷步驟**：
1. 檢查備份排程設定
2. 驗證儲存空間
3. 查看備份日誌

**解決方案**：
- 確保儲存空間足夠
- 檢查備份目的地權限
- 手動觸發備份測試

## 資源問題

### 記憶體不足

**症狀**：OOM Killer 終止容器

**診斷步驟**：
```bash
# 使用 coolify_server_resources 檢查
# 查看 dmesg 日誌
dmesg | grep -i "out of memory"
```

**解決方案**：
- 增加容器記憶體限制
- 優化應用程式記憶體使用
- 升級伺服器

### 磁碟空間不足

**症狀**：部署或寫入失敗

**診斷步驟**：
```bash
df -h
docker system df
```

**解決方案**：
```bash
# 清理 Docker 資源
docker system prune -a
docker volume prune

# 清理日誌
truncate -s 0 /var/lib/docker/containers/*/*-json.log
```

### CPU 使用率過高

**症狀**：應用響應緩慢

**診斷步驟**：
1. 使用 `coolify_server_resources` 檢查
2. 識別高 CPU 容器
3. 分析應用程式效能

**解決方案**：
- 優化程式碼
- 增加實例數量
- 升級伺服器

## 認證問題

### API Token 無效

**症狀**：401 Unauthorized 錯誤

**解決方案**：
1. 重新生成 API Token
2. 更新環境變數
3. 重啟 Claude Code

### 權限不足

**症狀**：403 Forbidden 錯誤

**解決方案**：
1. 檢查 Token 權限範圍
2. 確認資源存取權限
3. 聯繫管理員

## 常見錯誤訊息

| 錯誤 | 原因 | 解決方案 |
|------|------|----------|
| `Connection refused` | 服務未運行 | 啟動服務 |
| `Name resolution failed` | DNS 問題 | 檢查網路設定 |
| `Permission denied` | 權限不足 | 檢查檔案權限 |
| `No space left on device` | 磁碟滿 | 清理空間 |
| `Container killed` | OOM | 增加記憶體 |
| `Build context too large` | .dockerignore 缺失 | 新增 .dockerignore |

## 診斷工具使用

### 快速診斷流程

1. **概覽檢查**
   ```
   coolify_infrastructure_overview
   ```

2. **問題掃描**
   ```
   coolify_scan_for_issues
   ```

3. **特定診斷**
   ```
   coolify_diagnose_application app_id
   coolify_diagnose_server server_id
   ```

4. **日誌分析**
   ```
   coolify_get_application_logs app_id lines=500
   ```

### 收集診斷資訊

報告問題時，收集以下資訊：
- Coolify 版本
- 錯誤訊息和堆疊追蹤
- 應用程式日誌
- 伺服器資源狀態
- 相關設定檔案
