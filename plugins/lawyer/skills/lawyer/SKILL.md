---
name: lawyer
description: This skill should be used when working on the lawyer-app project (劉尹惠律師事務所), including Payload CMS content management, Next.js development, Coolify deployment, E2E testing, or content migration. Use when user mentions "lawyer app", "律師網站", "Payload CMS articles", "lawyer deployment", "lawyer testing", "lawyer staging", or "lawyer middleware".
---

# Lawyer App 開發指南

劉尹惠律師事務所（Liu Yin-Hui Law Office）官方網站，基於 Next.js 16 + Payload CMS 3.x 的全端架構。版本 1.3.0。

## 技術架構

| 層級 | 技術 | 說明 |
|------|------|------|
| 前端 | Next.js 16.1.6 App Router | `app/(frontend)/` route group，Turbopack |
| CMS | Payload CMS 3.75.0 | `app/(payload)/` route group → `/admin` |
| 樣式 | Tailwind CSS 4 + SCSS | `tailwind.config.ts` + `custom.scss` admin 主題 |
| 字體 | LXGW WenKai TC | Google Fonts via `FontLoader` 元件 |
| DB | PostgreSQL | Coolify 託管（Staging 5447 / Production 5448） |
| 圖片 | S3 Storage（MinIO） | 可選，`@payloadcms/storage-s3` conditional spread |
| 深色模式 | next-themes | `ThemeProvider` + `ThemeToggle` |

## 開發命令

```bash
bun dev              # 啟動開發伺服器 localhost:3000（Turbopack）
bun build            # 生產環境建置
bun lint             # ESLint 檢查
bun run typecheck    # TypeScript 型別檢查（tsc --noEmit）
bun run test         # Vitest 單元測試（注意：不是 bun test）
bun run test:e2e     # Playwright E2E 測試（需在 :3001 啟動 server）
bun run generate:types  # 重新產生 payload-types.ts
```

## 專案結構

```
app/(frontend)/        # 公開前台路由
  layout.tsx           # 根 layout（字體、metadata、ThemeProvider、JSON-LD）
  page.tsx             # 首頁（7 個業務區塊 + 最新文章）
  articles/            # 文章列表 + [slug] 詳情頁
  robots.ts            # SEO robots.txt（Staging 回傳 Disallow）
  sitemap.ts           # 動態 sitemap
  api/og/route.tsx     # OG 圖片生成
app/(payload)/         # Payload CMS Admin → /admin
  layout.tsx           # Admin layout（CSS imports）
  custom.scss          # Admin 主題覆蓋（sage green 設計系統，598 行）
collections/           # 4 個 Payload Collections
components/            # 15 個共用元件
hooks/                 # useScrollY、useIsClient
lib/payload/queries.ts # Server-side Local API 查詢（8 個函式，皆有 try-catch）
middleware.ts          # Staging HTTP Basic Auth（STAGING=true 啟用）
migrations/            # DB 遷移檔（prodMigrations 自動執行）
scripts/               # migrate-articles.ts、create-test-user.ts
tests/e2e/             # 7 個 Playwright spec files（另含 admin-setup）
```

## Payload CMS Collections

| Collection | 說明 | 特殊行為 |
|-----------|------|---------|
| **Articles** | 文章管理 | `beforeValidate` hook 自動 slug；fields: title, slug, content, excerpt, featureImage, tags, status, publishedAt, metaDescription |
| **Tags** | 標籤分類 | name, slug |
| **Media** | 媒體檔案 | 3 種 image sizes: thumbnail(400x300), card(768x512), hero(1200x630) |
| **Users** | 管理員 | auth: true |

Slug 自動生成保留 CJK 字元（`\u4e00-\u9fff`），在 `beforeValidate` hook 伺服器端執行。

## 設計系統

| Token | 值 | 用途 |
|-------|---|------|
| `primary` | #6A7E72 (Muted Sage Green) | 標題、CTA |
| `background-light` | #E6E4E1 (Pale Pebble Gray) | 淺色背景 |
| `background-dark` | #2C3530 (Deep Forest Green) | 深色背景 |
| `text-dark` | #333331 | 淺色模式文字 |
| `text-light` | #E8E6E3 (Warm Cream) | 深色模式文字 |
| `text-muted` | #A8A5A0 | 次要文字 |

自訂 typography classes（`globals.css` 797 行）：`.text-label`、`.font-display`、`.prose-elegant`、`.elegant-quote`、`.stat-number`。9 個動畫 keyframes。

## 部署工作流

### 環境對應

| 環境 | 分支 | 網址 | Coolify UUID |
|------|------|------|-------------|
| Production | `main` | https://lawyer.jurislm.com | `xckckgo84kwk8044wockkoso` |
| Staging | `develop` | https://lawyer-dev.jurislm.com | `jgcwwwc04sk04skc84488wk8` |

Push to branch → Coolify 自動部署（Nixpacks + bun）。

### 必要環境變數

```
DATABASE_URL=postgresql://user:pass@host:5432/db  # Build-time 必須可用
PAYLOAD_SECRET=<32+ 字元隨機字串>
STAGING=true                                       # 僅 Staging
```

### Staging 保護（三層）

1. **HTTP Basic Auth**（`middleware.ts`）— `STAGING_USER`/`STAGING_PASSWORD` env vars 設定
2. **robots.ts** — `Disallow: /`
3. **X-Robots-Tag** header — `noindex, nofollow, noarchive`（`next.config.ts`）

Middleware 排除：`/api/*`（webhook）、`robots.txt`、`sitemap.xml`、靜態資源。`Basic` scheme 比較為 case-insensitive（RFC 7617），`atob()` 有 try-catch 防 malformed base64。

### 關鍵注意事項

- `DATABASE_URL` 中的 `!` 必須 URL-encode 為 `%21`
- 內部連線使用 DB UUID 作為 hostname
- `push: true` 僅 dev mode 有效；Production 使用 `prodMigrations`
- S3 配置為可選 — **不要設 placeholder env vars**（會觸發 importMap 白屏）
- Cloudflare WAF Free plan 不支援 `cf.bot_management.verified_bot`

### Database Migration

```bash
bun -e "
import { getPayload } from 'payload';
import config from './payload.config.ts';
const payload = await getPayload({ config });
await payload.db.createMigration({ payload, migrationName: 'describe_change' });
process.exit(0);
"
```

`payload` CLI 在 Node.js 24 + bun 下無法使用。Commit migration 檔案 + `migrations/index.ts`，push 後 Payload 啟動時自動執行。

## E2E 測試

- **框架**: Playwright 1.58.2
- **測試規模**: 7 個 spec files + 1 個 setup 檔（admin-setup, admin, frontend 三個 project）
- **認證共享**: `admin-setup` 登入一次 → `storageState` 共享給 `admin` tests
- **Base URL**: `http://localhost:3001`

## GitHub Actions

| Workflow | 觸發 | 用途 |
|----------|------|------|
| `release.yml` | push to main | Release Please 自動版本管理 |
| `claude.yml` | PR/Issue @claude | Claude Code Review |

## 參考文件

- **`references/payload-cms.md`** — Payload CMS 配置模式、importMap 白屏陷阱、Admin CSS 修復
- **`references/deployment.md`** — Coolify 部署流程、DB 配置、常見問題診斷
- **`references/testing.md`** — E2E 測試架構、7 個已知陷阱與解法
