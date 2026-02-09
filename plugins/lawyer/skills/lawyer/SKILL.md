---
name: lawyer
description: This skill should be used when working on the lawyer-app project (劉尹惠律師事務所), including Payload CMS content management, Next.js development, Coolify deployment, E2E testing, or content migration. Use when user mentions "lawyer app", "律師網站", "Payload CMS articles", "lawyer deployment", or "lawyer testing".
---

# Lawyer App 開發指南

劉尹惠律師事務所（Liu Yin-Hui Law Office）官方網站，基於 Next.js 16 + Payload CMS 3.x 的全端架構。

## 技術架構

| 層級 | 技術 | 說明 |
|------|------|------|
| 前端 | Next.js 16 App Router | `app/(frontend)/` route group |
| CMS | Payload CMS 3.x | `app/(payload)/` route group → `/admin` |
| 樣式 | Tailwind CSS + next-themes | 自訂設計系統，支援深色模式 |
| 字體 | Noto Serif + Noto Sans | Google Fonts via `next/font` |
| DB | PostgreSQL | Coolify 託管，port 5447 |
| 圖片 | S3 Storage（MinIO） | 待部署，目前可選 |

## 開發命令

```bash
bun dev      # 啟動開發伺服器 localhost:3000
bun build    # 生產環境建置
bun lint     # ESLint 檢查
bun run test # Vitest 單元測試（注意：不是 bun test）
```

### E2E 測試

```bash
# 先建立測試帳號（首次使用）
bun run scripts/create-test-user.ts

# 執行所有 E2E 測試（90 項）
bunx playwright test

# 執行特定測試檔
bunx playwright test tests/e2e/admin-articles.spec.ts

# 帶 UI 模式
bunx playwright test --ui
```

## Payload CMS Collections

| Collection | 說明 | 特殊行為 |
|-----------|------|---------|
| **Articles** | 文章管理 | `beforeValidate` hook 自動從 title 生成 slug |
| **Tags** | 標籤分類 | 用於文章篩選 |
| **Media** | 媒體檔案 | S3 或本地儲存 |
| **Users** | 使用者管理 | 管理員帳號 |

### Slug 自動生成

Articles 的 slug 由 `beforeValidate` hook 在伺服器端自動產生：

```typescript
function slugify(text: string): string {
  return text
    .toLowerCase()
    .trim()
    .replace(/\s+/g, '-')
    .replace(/[^\w\u4e00-\u9fff\u3400-\u4dbf-]/g, '')  // 保留中文字元
    .replace(/-+/g, '-')
    .replace(/^-|-$/g, '')
}
```

### 管理後台存取

- **URL**: `https://<domain>/admin`
- **測試帳號**: 由 `scripts/create-test-user.ts` 建立
- **認證方式**: Email + Password

## 設計系統

### 色彩

| Token | 值 | 用途 |
|-------|---|------|
| `primary` | #6A7E72 (Muted Sage Green) | 標題、CTA |
| `background-light` | #E6E4E1 (Pale Pebble Gray) | 淺色背景 |
| `background-dark` | #363634 (Dark Stone) | 深色背景 |
| `text-dark` | #333331 | 淺色模式文字 |
| `text-light` | #F2F0ED | 深色模式文字 |

### 深色模式

使用 `next-themes` 的 `ThemeProvider`：
- `<html>` 需要 `suppressHydrationWarning`
- Body 需要 `dark:bg-background-dark dark:text-text-light` 類別
- Navigation 中包含 `ThemeToggle` 元件

## 部署工作流

### 環境對應

| 環境 | 分支 | 網址 | Coolify UUID |
|------|------|------|-------------|
| Production | `main` | https://lawyer.jurislm.com | `xckckgo84kwk8044wockkoso` |
| Staging | `develop` | https://lawyer-staging.jurislm.com | `jgcwwwc04sk04skc84488wk8` |

### 部署步驟

1. **Push to branch** → Coolify auto-deploy
2. **確認環境變數**（首次部署必須）：
   ```bash
   DATABASE_URI=postgresql://payload:LawyerCMS2026%21@<db-uuid>:5432/lawyer_cms
   PAYLOAD_SECRET=<random-secret>
   NEXT_PUBLIC_SERVER_URL=https://lawyer.jurislm.com
   ```
3. **驗證部署**：
   - 前台：`https://<domain>/`
   - 後台：`https://<domain>/admin`
   - API：`https://<domain>/api/articles`

### Staging 保護

Staging 環境（`STAGING=true`）有三層保護：

1. **HTTP Basic Auth**（`middleware.ts`）— 帳號 `admin` / 密碼 `staging2026`（可透過 `STAGING_USER`/`STAGING_PASSWORD` env var 覆蓋）
2. **robots.ts** — 回傳 `Disallow: /`
3. **X-Robots-Tag** header — `noindex, nofollow, noarchive`（`next.config.ts`）

Middleware 排除路徑：`/api/*`（webhook）、`robots.txt`、`sitemap.xml`、靜態資源。

### 關鍵注意事項

- `DATABASE_URI` 中的 `!` 必須 URL-encode 為 `%21`
- 內部連線使用 DB UUID 作為 hostname（`dow48808w0wggws8wo8ck8g4:5432`）
- `DATABASE_URI` 必須在 build-time 可用（Payload 在 `next build` 時執行 migration）
- S3 配置為可選（payload.config.ts 使用 conditional spread）
- Cloudflare WAF Free plan 不支援 `cf.bot_management.verified_bot` 欄位 — 需靠 middleware Basic Auth 擋爬蟲

## E2E 測試架構

### 配置

- **框架**: Playwright
- **配置檔**: `playwright.config.ts`
- **測試數量**: 90 項（88 passed + 2 flaky with retry）
- **Retry**: 本地 1 次，CI 2 次

### Project 結構

| Project | 用途 | 認證 |
|---------|------|------|
| `admin-setup` | 登入並儲存 session | 產生 `storageState` |
| `admin` | 後台功能測試 | 使用共享 `storageState` |
| `frontend` | 前台功能測試 | 無需認證 |

### 測試檔案

| 檔案 | 涵蓋範圍 |
|------|---------|
| `admin-setup.ts` | 登入 + 儲存認證狀態 |
| `admin-auth.spec.ts` | 登入/登出/未授權存取 |
| `admin-dashboard.spec.ts` | Dashboard、側邊欄、導覽 |
| `admin-articles.spec.ts` | 文章 CRUD、slug 生成 |
| `admin-tags-media-users.spec.ts` | 標籤、媒體、使用者管理 |
| `frontend-homepage.spec.ts` | 首頁元件、FloatingCTA |
| `frontend-articles.spec.ts` | 文章列表、標籤篩選、詳情頁 |
| `frontend-navigation-theme-seo.spec.ts` | 導覽、深色模式、SEO |

### 已知陷阱

1. **不要硬編碼 baseURL** — 使用相對路徑 `/articles`，依賴 `playwright.config.ts`
2. **測試帳密用環境變數** — `process.env.E2E_ADMIN_EMAIL || 'fallback'`
3. **storageState 目錄** — 寫入前需 `mkdirSync('tests/e2e/.auth', { recursive: true })`
4. **catch 中的 exit code** — 錯誤時 `process.exit(1)` 而非 `process.exit(0)`
5. **Slug 測試** — 使用 `Date.now()` 避免 unique constraint 衝突
6. **ThemeToggle 測試** — 需 `waitForLoadState('networkidle')` 等待 hydration

## 內容遷移（Ghost → Payload）

### 遷移腳本

`scripts/migrate-articles.ts` — 從 Ghost Content API 匯入文章到 Payload CMS

### 前置條件

1. Ghost CMS 仍在運行（提供 Content API）
2. Payload CMS 已部署（Database + Admin 可用）
3. 環境變數已設定

### 遷移流程

1. 從 Ghost API 取得所有文章
2. 轉換為 Payload Article 格式
3. 處理標籤對應（建立或匹配現有 Tag）
4. 處理圖片（下載 + 上傳到 Payload Media）
5. 建立 Article 記錄

## 常見問題

### Admin CSS 不載入（Turbopack）

Payload 3.x + Next.js 16 Turbopack 下需要手動載入：
1. `custom.scss` 中 `@import '@payloadcms/ui/scss/app.scss'`
2. `layout.tsx` 中 `import '@payloadcms/next/css'`（不能用 SCSS `@import`）

### robots.ts / sitemap.ts 404

這些檔案必須在 `app/` 根目錄，不能在 `app/(frontend)/` route group 內。

### Payload CLI migration 失敗

`payload migrate` 使用 tsx，在 Node.js 24 下無法解析 `.ts` import。
解法：`bun -e "import ... from './payload.config.ts'"` 直接執行。

## 參考文件

- **`references/payload-cms.md`** — Payload CMS 配置模式與陷阱
- **`references/deployment.md`** — Coolify 部署完整流程
- **`references/testing.md`** — E2E 測試詳細指南
