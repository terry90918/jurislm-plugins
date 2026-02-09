# Payload CMS 配置模式與陷阱

## Collection 設計

### Articles Collection

```typescript
// collections/Articles.ts 關鍵結構
{
  slug: 'articles',
  admin: { useAsTitle: 'title' },
  hooks: {
    beforeValidate: [
      ({ data }) => {
        // 伺服器端自動生成 slug
        if (data && data.title && !data.slug) {
          data.slug = slugify(data.title)
        }
        return data
      },
    ],
  },
  fields: [
    { name: 'title', type: 'text', required: true },
    { name: 'slug', type: 'text', unique: true, admin: { position: 'sidebar' } },
    { name: 'content', type: 'richText' },
    { name: 'excerpt', type: 'textarea' },
    { name: 'featuredImage', type: 'upload', relationTo: 'media' },
    { name: 'tags', type: 'relationship', relationTo: 'tags', hasMany: true },
    { name: 'status', type: 'select', options: ['draft', 'published'] },
    { name: 'publishedAt', type: 'date' },
  ],
}
```

### Slug 生成函式

支援中文字元的 slugify：

```typescript
function slugify(text: string): string {
  return text
    .toLowerCase()
    .trim()
    .replace(/\s+/g, '-')
    .replace(/[^\w\u4e00-\u9fff\u3400-\u4dbf-]/g, '')  // 保留 CJK 字元
    .replace(/-+/g, '-')
    .replace(/^-|-$/g, '')
}
```

**注意**：slug 生成在 `beforeValidate` hook 中執行，不是前端 JavaScript。客戶端填寫 title 後不會立即看到 slug 欄位填入值，而是在按下 Save 後由伺服器處理。

## S3 Storage 配置（可選）

```typescript
// payload.config.ts
import { s3Storage } from '@payloadcms/storage-s3'

export default buildConfig({
  plugins: [
    // 使用 conditional spread 讓 S3 可選
    ...(process.env.S3_ACCESS_KEY_ID
      ? [
          s3Storage({
            collections: { media: true },
            bucket: requireEnv('S3_BUCKET'),
            config: {
              credentials: {
                accessKeyId: requireEnv('S3_ACCESS_KEY_ID'),
                secretAccessKey: requireEnv('S3_SECRET_ACCESS_KEY'),
              },
              endpoint: process.env.S3_ENDPOINT,
              region: process.env.S3_REGION || 'us-east-1',
              forcePathStyle: true, // MinIO 必須
            },
          }),
        ]
      : []),
  ],
})
```

**陷阱**：S3 未配置時，上傳功能 fallback 到本地檔案系統。生產環境應配置 S3 避免容器重建時丟失檔案。

### importMap 與 S3 Plugin 白屏陷阱（嚴重）

**問題**：Admin UI 完全白屏（HTML 載入但 React 不渲染任何內容）。

**根因**：
1. S3 plugin 使用 conditional spread，根據 `S3_ACCESS_KEY_ID` env var 是否 truthy 決定是否載入
2. S3 plugin 載入時會註冊 `@payloadcms/storage-s3/client#S3ClientUploadHandler` 客戶端元件
3. `importMap.js` 是在 `next build` 時生成的，必須包含所有活躍 plugin 的客戶端元件
4. **如果 build 環境的 S3 env var 狀態與 runtime 不同**（例如 build 時沒設 → importMap 不含 S3；runtime 有設 → S3 plugin 載入），React 找不到元件 → **整個 Admin UI 靜默失敗**

**觸發場景**：
- 本地 build（S3_ACCESS_KEY_ID 為空）→ 部署到 Coolify（S3_ACCESS_KEY_ID=placeholder）
- importMap.js 不含 S3 元件 → S3 plugin 嘗試註冊找不到的元件 → 白屏

**解法**：
1. **不要在 Coolify 設定 placeholder S3 env vars**——留空或完全不設定
2. 確保 build 環境與 runtime 環境的 S3 env var 狀態一致
3. 如果需要 S3，在 build time 也要有正確的 S3 env vars，或在 build 後執行 `payload generate:importmap`

**診斷方法**：
```bash
# 檢查 server log 是否有 importMap 警告
# 關鍵錯誤：
# "getFromImportMap: PayloadComponent not found in importMap"
# "@payloadcms/storage-s3/client#S3ClientUploadHandler"

# 檢查 RSC payload 是否有 redirect 而非實際 HTML
curl -s https://your-domain.com/admin | grep "NEXT_REDIRECT"
```

## Admin UI CSS 問題（Turbopack）

### 問題

Payload 3.75.0 + Next.js 16.1.6 + Turbopack 下，Admin UI 的 CSS 不會自動載入，導致後台完全無樣式。

### 解法

需要手動載入兩個 CSS 來源：

**1. CSS 變數（`:root` 定義）**

在 `app/(payload)/custom.scss`：
```scss
@import '@payloadcms/ui/scss/app.scss';

// 自訂覆蓋（可選）
:root {
  --theme-text: #333331;
  --font-body: 'LXGW WenKai', serif;
}
```

**2. Template Layout CSS（304KB）**

在 `app/(payload)/layout.tsx`：
```typescript
import '@payloadcms/next/css'  // 必須用 JS import，不能用 SCSS @import
```

### 為什麼不能用 SCSS @import

`@payloadcms/next/css` 指向已編譯的 CSS 檔案，包含 Sass 不支援的語法。用 SCSS `@import` 會報 `Expected digit` 錯誤。

## Database Migration

### 問題

`payload migrate` 使用 tsx loader，在 Node.js 24 + `moduleResolution: "bundler"` 下無法解析 `.ts` 相對 import。

### 解法

使用 bun 直接執行：

```bash
bun -e "
import config from './payload.config.ts'
import { getPayload } from 'payload'
const payload = await getPayload({ config })
await payload.db.migrate()
console.log('Migration complete')
process.exit(0)
"
```

### TypeScript 配置

`tsconfig.json` 需要：
```json
{
  "compilerOptions": {
    "allowImportingTsExtensions": true
  }
}
```

因為 `payload.config.ts` 中的 collection import 需要 `.ts` 後綴（for bun/Node 24 ESM）。

## 資料查詢（前端）

### 查詢封裝

```typescript
// lib/payload/queries.ts
import { getPayload } from 'payload'
import config from '@/payload.config'

export async function getArticles(options?: {
  limit?: number
  page?: number
  tag?: string
}) {
  const payload = await getPayload({ config })
  return payload.find({
    collection: 'articles',
    where: {
      status: { equals: 'published' },
      ...(options?.tag && {
        'tags.name': { equals: options.tag },
      }),
    },
    sort: '-publishedAt',
    limit: options?.limit || 10,
    page: options?.page || 1,
  })
}
```

### Server Component 使用

```typescript
// app/(frontend)/articles/page.tsx
import { getArticles } from '@/lib/payload/queries'

export default async function ArticlesPage({ searchParams }) {
  const { docs, totalPages } = await getArticles({
    tag: searchParams.tag,
    page: Number(searchParams.page) || 1,
  })
  // render...
}
```
