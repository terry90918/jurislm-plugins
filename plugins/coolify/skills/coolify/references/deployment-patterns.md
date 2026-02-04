# Coolify 部署模式與最佳實踐

## 部署類型

### Nixpacks（推薦）

Nixpacks 是 Coolify 預設的建置系統，自動偵測專案類型並建置。

支援語言/框架：
- Node.js / Bun / Deno
- Python / Django / FastAPI
- Ruby / Rails
- Go
- Rust
- PHP / Laravel
- .NET / C#
- Java / Kotlin
- Elixir / Phoenix

優點：
- 零配置
- 快速建置
- 小型映像檔

### Dockerfile

適用於需要完全控制建置過程的專案。

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build
EXPOSE 3000
CMD ["npm", "start"]
```

### Docker Compose

適用於多容器應用程式。

```yaml
services:
  web:
    build: .
    ports:
      - "3000:3000"
    depends_on:
      - db
  db:
    image: postgres:16
```

### 靜態網站

適用於純前端專案。

設定：
- `build_command`: `npm run build`
- `publish_directory`: `dist` 或 `build`

## 部署策略

### 滾動更新（Rolling Update）

1. 啟動新版本容器
2. 健康檢查通過
3. 移除舊版本容器

設定健康檢查：
```
health_check_path: /api/health
health_check_interval: 30
health_check_timeout: 5
```

### 藍綠部署（Blue-Green）

1. 準備兩個環境（藍/綠）
2. 部署新版本到非活躍環境
3. 測試通過後切換域名
4. 保留舊環境以便回滾

實作步驟：
1. 建立兩個應用程式：`app-blue`、`app-green`
2. 主域名指向當前活躍環境
3. 部署到非活躍環境並測試
4. 更新域名指向

### 金絲雀部署（Canary）

1. 部署新版本到小比例實例
2. 監控錯誤率和效能
3. 逐步增加比例
4. 完全切換或回滾

需要負載平衡器支援。

## 環境管理

### 開發/測試/生產

建議為每個環境建立獨立專案：

```
projects/
├── myapp-dev/
│   ├── app
│   └── db
├── myapp-staging/
│   ├── app
│   └── db
└── myapp-prod/
    ├── app
    └── db
```

### 環境變數管理

敏感資料使用環境變數：
```bash
DATABASE_URL=postgresql://user:pass@db:5432/app
REDIS_URL=redis://redis:6379
SECRET_KEY=your-secret-key
```

不要：
- 硬編碼在程式碼中
- 提交到 Git
- 在日誌中輸出

### 分支部署

自動為 PR 建立預覽環境：

1. 建立 Webhook（GitHub/GitLab）
2. 設定分支規則
3. PR 建立時自動部署
4. PR 合併後自動刪除

## 資源配置

### 記憶體限制

根據應用類型設定：

| 類型 | 建議記憶體 |
|------|-----------|
| 靜態網站 | 128MB |
| Node.js API | 256MB-512MB |
| Next.js | 512MB-1GB |
| Python/Django | 256MB-512MB |
| Java | 512MB-2GB |

### CPU 限制

- 小型應用：0.5 CPU
- 中型應用：1-2 CPU
- 大型應用：2-4 CPU

### 副本數量

根據流量和可用性需求：
- 開發環境：1 副本
- 生產環境：2-3 副本（高可用性）

## 域名與 SSL

### 自定義域名

1. 在 DNS 設定 A/CNAME 記錄
2. 在 Coolify 新增域名
3. 等待 SSL 證書簽發（Let's Encrypt）

### Wildcard 域名

適用於多租戶應用：
```
*.app.example.com → 主應用
tenant1.app.example.com
tenant2.app.example.com
```

需要 DNS 驗證以取得 Wildcard 證書。

### SSL 設定

- **自動**：Let's Encrypt 自動簽發和更新
- **手動**：上傳自己的證書
- **禁用**：僅限內部服務

## 監控與日誌

### 健康檢查

設定應用程式健康檢查端點：

```javascript
// Express.js
app.get('/health', (req, res) => {
  res.json({ status: 'ok', timestamp: new Date() });
});
```

### 日誌最佳實踐

- 使用結構化日誌（JSON）
- 包含請求 ID 以便追蹤
- 設定適當的日誌級別
- 避免記錄敏感資料

### 效能監控

整合監控工具：
- Prometheus + Grafana
- Uptime Kuma（內建支援）
- Sentry（錯誤追蹤）

## 備份策略

### 資料庫備份

設定自動備份：
```
頻率：每日
保留期：30 天
時間：凌晨 3:00（低流量時段）
```

### 應用程式備份

- 儲存在 Git（程式碼）
- 環境變數另外備份
- 上傳檔案使用 S3 或類似服務

### 災難復原

1. 文件化所有設定
2. 定期測試復原流程
3. 保留多個區域的備份
4. 設定 RTO/RPO 目標

## 安全最佳實踐

### 網路安全

- 資料庫不對外暴露
- 使用內部網路通訊
- 設定防火牆規則
- 啟用 DDoS 保護

### 應用安全

- 保持依賴更新
- 定期掃描漏洞
- 實施最小權限原則
- 加密敏感資料

### 存取控制

- 使用最小權限 API Token
- 定期輪換密碼和 Token
- 啟用雙因素認證
- 審計重要操作
