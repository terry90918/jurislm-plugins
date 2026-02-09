# Coolify 部署完整流程

## 環境對照

| 項目 | Production | Staging |
|------|-----------|---------|
| 分支 | `main` | `develop` |
| 網址 | https://lawyer.jurislm.com | https://lawyer-dev.jurislm.com |
| Coolify UUID | `xckckgo84kwk8044wockkoso` | `jgcwwwc04sk04skc84488wk8` |
| 自動部署 | push to main | push to develop |

## 資料庫

### Production DB

| 項目 | 值 |
|------|---|
| DB UUID | `v80s40gk4g4gsosk8c04s8c0` |
| Public Port | 5448 |
| External URL | `postgres://payload:<password>@46.225.58.202:5448/lawyer_cms` |
| Internal URL | `postgres://payload:<password>@v80s40gk4g4gsosk8c04s8c0:5432/lawyer_cms` |

### Staging DB

| 項目 | 值 |
|------|---|
| DB UUID | `dow48808w0wggws8wo8ck8g4` |
| Public Port | 5447 |
| External URL | `postgres://payload:<password>@46.225.58.202:5447/lawyer_cms` |
| Internal URL | `postgres://payload:<password>@dow48808w0wggws8wo8ck8g4:5432/lawyer_cms` |

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
DATABASE_URL=postgresql://payload:<password>@<db-uuid>:5432/lawyer_cms
PAYLOAD_SECRET=<至少 32 字元隨機字串>
```

**Staging 專用**：
```bash
STAGING=true
STAGING_USER=<username>
STAGING_PASSWORD=<password>
```

**可選變數**（S3 Storage）：
```bash
S3_BUCKET=lawyer-media
S3_ACCESS_KEY_ID=<minio-access-key>
S3_SECRET_ACCESS_KEY=<minio-secret-key>
S3_ENDPOINT=https://minio.jurislm.com
S3_REGION=us-east-1
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
- [ ] robots.txt 可存取
- [ ] sitemap.xml 可存取

## Staging 保護

Staging 環境（`STAGING=true`）有三層保護：

### 1. HTTP Basic Auth（middleware.ts）

透過 `STAGING_USER`/`STAGING_PASSWORD` 環境變數設定。Middleware 排除：`/api/*`、`robots.txt`、`sitemap.xml`、靜態資源。

### 2. robots.ts

`STAGING=true` 時回傳 `Disallow: /`：

```typescript
export default function robots() {
  if (process.env.STAGING === 'true') {
    return { rules: { userAgent: '*', disallow: '/' } }
  }
  // production rules...
}
```

### 3. X-Robots-Tag Header

`next.config.ts` 中 `STAGING=true` 時加入 `noindex, nofollow, noarchive` header。

## Production Database Migration

Production 使用 `prodMigrations`，伺服器啟動時自動執行。**`push: true` 僅在 `next dev` 有效。**

### 建立新 Migration

```bash
# payload CLI 在 Node.js 24 + bun 下壞掉 — 使用程式化 API：
bun -e "
import { getPayload } from 'payload';
import config from './payload.config.ts';
const payload = await getPayload({ config });
await payload.db.createMigration({ payload, migrationName: 'describe_change' });
process.exit(0);
"
```

前置條件：`.env.local` 須包含 `PAYLOAD_SECRET` 與 `DATABASE_URL`，且 PostgreSQL 可連線。

產生的 migration 檔案 + 更新後的 `migrations/index.ts` 必須一併 commit 並 push。

## 常見部署問題

### Build 失敗：DATABASE_URL 未設定

**症狀**：`next build` 時 Payload 無法連接資料庫

**原因**：Payload CMS 在 build time 需要 DB 連線來執行 migration 和生成 schema

**解法**：在 Coolify 環境變數中確認 `DATABASE_URL` 在 build-time 可用

### Build 失敗：記憶體不足

**症狀**：`FATAL ERROR: JavaScript heap out of memory`

**解法**：加入 `NODE_OPTIONS=--max-old-space-size=4096`

### Admin UI 空白（CSS 問題）

**症狀**：Admin 頁面載入但無樣式

**原因**：Turbopack 不自動載入 Payload CSS

**解法**：見 `references/payload-cms.md` → Admin UI CSS 問題

### Admin UI 完全白屏（importMap 問題）

**症狀**：Admin 頁面 HTML 載入（200 OK），但 React 不渲染任何元素。

**原因**：S3 plugin 的 env var 在 build time 與 runtime 狀態不一致。

**診斷**：
1. 檢查 server log：`getFromImportMap: PayloadComponent not found`
2. `curl -s https://site.com/admin | grep "NEXT_REDIRECT"`

**解法**：
1. 移除 Coolify 中所有 S3 placeholder env vars
2. 或確保 build time 也有正確的 S3 env vars
3. 見 `references/payload-cms.md` → importMap 與 S3 Plugin 白屏陷阱

### 文章圖片 404

**原因**：S3 未配置，容器重建導致本地檔案丟失

**解法**：配置 S3 Storage（MinIO）

## Nixpacks Bun 配置

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

1. **package-lock.json 導致 Nixpacks 用 npm**：Nixpacks 偵測到 `package-lock.json` 會自動使用 `npm ci`。確保 repo 中沒有（已加入 `.gitignore`）。

2. **NIXPACKS_BUILD_CMD env var 解析錯誤**：`NIXPACKS_BUILD_CMD=bun run build` 會導致 CLI 誤將 `run` 當作 argument。**不要用 NIXPACKS_*_CMD env vars**。

## 回滾流程

1. 在 Coolify 中找到上一次成功的部署
2. `coolify_deploy <uuid> force=true` 重新部署
3. 或 `git revert` + push 觸發新部署
