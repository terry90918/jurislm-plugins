# Coolify 部署完整流程

## 環境對照

| 項目 | Production | Staging |
|------|-----------|---------|
| 分支 | `main` | `develop` |
| 網址 | https://lawyer.jurislm.com | https://lawyer-staging.jurislm.com |
| Coolify UUID | `xckckgo84kwk8044wockkoso` | `jgcwwwc04sk04skc84488wk8` |
| 自動部署 | push to main | push to develop |

## 資料庫

| 項目 | 值 |
|------|---|
| DB UUID | `dow48808w0wggws8wo8ck8g4` |
| DB Type | PostgreSQL |
| Public Port | 5447 |
| External URL | `postgres://payload:LawyerCMS2026%21@46.225.58.202:5447/lawyer_cms` |
| Internal URL | `postgres://payload:LawyerCMS2026%21@dow48808w0wggws8wo8ck8g4:5432/lawyer_cms` |

**密碼注意**：`!` 必須 URL-encode 為 `%21`，否則 pg driver 認證失敗。

## 首次部署檢查清單

### 1. 建立 Database

```
coolify_create_database
  type: postgresql
  name: lawyer-cms-db
  postgres_user: payload
  postgres_password: <password>
  postgres_db: lawyer_cms
```

### 2. 設定環境變數

**必要變數**：
```bash
DATABASE_URI=postgresql://payload:LawyerCMS2026%21@dow48808w0wggws8wo8ck8g4:5432/lawyer_cms
PAYLOAD_SECRET=<至少 32 字元隨機字串>
NEXT_PUBLIC_SERVER_URL=https://lawyer.jurislm.com
```

**可選變數**（S3 Storage）：
```bash
S3_BUCKET=lawyer-media
S3_ACCESS_KEY=<minio-access-key>
S3_SECRET_KEY=<minio-secret-key>
S3_ENDPOINT=https://minio.jurislm.com
S3_REGION=us-east-1
```

**E2E 測試變數**：
```bash
E2E_ADMIN_EMAIL=e2e-test@jurislm.com
E2E_ADMIN_PASSWORD=E2ETest2026
```

### 3. 設定 FQDN

```
coolify_update_application
  uuid: <app-uuid>
  fqdn: https://lawyer.jurislm.com
```

### 4. 觸發部署

```
coolify_deploy <app-uuid>
```

### 5. 驗證

- [ ] 前台首頁載入：`https://lawyer.jurislm.com`
- [ ] Admin 後台載入：`https://lawyer.jurislm.com/admin`
- [ ] Admin CSS 正常顯示
- [ ] 文章列表 API：`https://lawyer.jurislm.com/api/articles`
- [ ] robots.txt 可存取：`https://lawyer.jurislm.com/robots.txt`
- [ ] sitemap.xml 可存取：`https://lawyer.jurislm.com/sitemap.xml`

## Staging 特殊設定

### 存取保護

Staging 環境應設定 HTTP Basic Auth 防止公開存取：
- Username: `admin`
- Password: `staging2026`

### SEO 保護

Staging 的 `robots.ts` 應回傳 Disallow：

```typescript
export default function robots() {
  const isProduction = process.env.NEXT_PUBLIC_SERVER_URL?.includes('lawyer.jurislm.com')
  if (!isProduction) {
    return { rules: { userAgent: '*', disallow: '/' } }
  }
  // production rules...
}
```

## 常見部署問題

### Build 失敗：DATABASE_URI 未設定

**症狀**：`next build` 時 Payload 無法連接資料庫

**原因**：Payload CMS 在 build time 需要 DB 連線來執行 migration 和生成 schema

**解法**：在 Coolify 環境變數中將 `DATABASE_URI` 標記為 build-time variable

### Build 失敗：記憶體不足

**症狀**：`FATAL ERROR: JavaScript heap out of memory`

**解法**：加入 `NODE_OPTIONS=--max-old-space-size=4096`

### Admin UI 空白（CSS 問題）

**症狀**：Admin 頁面載入但無樣式

**原因**：Turbopack 不自動載入 Payload CSS

**解法**：見 `references/payload-cms.md` → Admin UI CSS 問題

### Admin UI 完全白屏（importMap 問題）

**症狀**：Admin 頁面 HTML 載入（200 OK），但 React 不渲染任何元素。頁面完全空白，無 console error。

**原因**：S3 plugin 的 env var 在 build time 與 runtime 狀態不一致，導致 `importMap.js` 不包含 S3 客戶端元件，但 runtime S3 plugin 已載入並嘗試使用該元件。

**診斷**：
1. 檢查 server log 是否有 `getFromImportMap: PayloadComponent not found`
2. `curl -s https://site.com/admin | grep "NEXT_REDIRECT"` — 如果有 redirect 到 /admin/login 但 login 也白屏，則可能是 importMap 問題

**解法**：
1. 移除 Coolify 中所有 S3 placeholder env vars（確保 `S3_ACCESS_KEY_ID` 為空或不存在）
2. 或者確保 build time 也有正確的 S3 env vars
3. 見 `references/payload-cms.md` → importMap 與 S3 Plugin 白屏陷阱

### 文章圖片 404

**症狀**：文章的 featured image 顯示 404

**原因**：
1. S3 未配置（生產環境必須）
2. 容器重建導致本地檔案丟失

**解法**：配置 S3 Storage（MinIO）

## Nixpacks Bun 配置

Lawyer App 使用 bun 作為 package manager。Coolify 使用 Nixpacks 建置時需要正確配置。

### nixpacks.toml

```toml
[phases.setup]
nixPkgs = ["bun"]

[phases.install]
cmds = ["bun install"]

[phases.build]
cmds = ["bun run build"]

[start]
cmd = "bun run start"
```

### 常見陷阱

1. **package-lock.json 導致 Nixpacks 用 npm**：
   - Nixpacks 偵測到 `package-lock.json` 會自動使用 `npm ci`
   - 必須確保 repo 中沒有 `package-lock.json`（已加入 `.gitignore`）
   - 只保留 `bun.lock`（bun 自動管理）

2. **NIXPACKS_BUILD_CMD env var 解析錯誤**：
   - 設定 `NIXPACKS_BUILD_CMD=bun run build` 會導致 Nixpacks CLI 誤將 `run` 當作 CLI argument
   - 錯誤：`Found argument 'run' which wasn't expected`
   - **不要用 NIXPACKS_*_CMD env vars**，改用 `nixpacks.toml` 檔案

## 回滾流程

1. 在 Coolify 中找到上一次成功的部署
2. 使用 `coolify_deploy <uuid> force=true` 重新部署指定版本
3. 或 git revert + push 觸發新部署
