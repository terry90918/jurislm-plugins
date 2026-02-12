# E2E 測試詳細指南

## 測試架構

### 配置

```typescript
// playwright.config.ts 關鍵設定
{
  baseURL: 'http://localhost:3001',  // 測試用 port
  retries: process.env.CI ? 2 : 1,   // 本地 1 次 retry
  projects: [
    {
      name: 'admin-setup',
      testMatch: /admin-setup\.ts/,
      // 登入並儲存認證狀態
    },
    {
      name: 'admin',
      dependencies: ['admin-setup'],
      use: { storageState: 'tests/e2e/.auth/admin.json' },
      // 後台功能測試
    },
    {
      name: 'frontend',
      // 前台功能測試，無需認證
    },
  ],
}
```

### 認證共享機制

`admin-setup` project 負責登入並將 session 儲存到 `storageState` 檔案，`admin` project 的所有測試自動使用此認證狀態，避免每個測試都重新登入。

**重要**：寫入 storageState 前必須確保目錄存在：
```typescript
import { mkdirSync } from 'node:fs'
mkdirSync('tests/e2e/.auth', { recursive: true })
```

## 測試帳號管理

### 建立測試帳號

```bash
bun run scripts/create-test-user.ts
```

腳本使用環境變數：
```typescript
const email = process.env.E2E_ADMIN_EMAIL || 'e2e-test@jurislm.com'
const password = process.env.E2E_ADMIN_PASSWORD || 'E2ETest2026'
```

### 環境變數設定

`.env.example` 中的 E2E 相關變數：
```bash
# E2E Test Configuration
E2E_ADMIN_EMAIL=e2e-test@jurislm.com
E2E_ADMIN_PASSWORD=E2ETest2026
```

## 測試覆蓋範圍（7 個 spec files + 1 個 setup）

### Admin 測試（5 個檔案）

| 檔案 | 測試數 | 涵蓋 |
|------|--------|------|
| admin-setup.ts | 1 | 登入 + storageState |
| admin-auth.spec.ts | ~10 | 登入/登出/未授權 |
| admin-dashboard.spec.ts | ~15 | Dashboard、側邊欄 |
| admin-articles.spec.ts | ~15 | 文章 CRUD、slug 生成 |
| admin-tags-media-users.spec.ts | ~15 | 標籤、媒體、使用者 |

### Frontend 測試（3 個檔案）

| 檔案 | 測試數 | 涵蓋 |
|------|--------|------|
| frontend-homepage.spec.ts | ~15 | Hero、業務區、FloatingCTA |
| frontend-articles.spec.ts | ~10 | 文章列表、篩選、詳情 |
| frontend-navigation-theme-seo.spec.ts | ~10 | 導覽、深色模式、SEO |

## 已知陷阱與解法

### 1. 硬編碼 baseURL

**錯誤**：
```typescript
await page.goto('http://localhost:3001/articles')
```

**正確**：
```typescript
await page.goto('/articles')  // 依賴 playwright.config.ts 的 baseURL
```

### 2. Slug 測試 unique constraint

**問題**：使用固定 title 建立文章，第二次執行會因 slug unique constraint 失敗

**解法**：
```typescript
const uniqueId = Date.now()
const title = `E2E Slug Test ${uniqueId}`
```

### 3. 等待伺服器端操作

**問題**：`waitForTimeout(3000)` 不可靠

**解法**：
```typescript
// 等待 URL 變化（表示 save 完成且 redirect 到編輯頁）
await page.waitForURL(/\/admin\/collections\/articles\/\d+/, {
  timeout: 15000,
})
```

### 4. ThemeToggle hydration

**問題**：ThemeToggle 在 client hydration 完成前不可見

**解法**：
```typescript
await page.waitForLoadState('networkidle')
await expect(themeToggle).toBeVisible({ timeout: 10000 })
```

### 5. Tag 篩選 networkidle

**問題**：點選 tag filter 後頁面未重新載入完畢就開始斷言

**解法**：
```typescript
await tagLink.click()
await page.waitForLoadState('networkidle')
```

### 6. robots.txt 大小寫

**問題**：Payload 回傳 `User-Agent`（capital A），但某些測試用 `User-agent`

**解法**：用 `.toLowerCase()` 比較：
```typescript
const text = await response.text()
expect(text.toLowerCase()).toContain('user-agent')
```

### 7. catch 中的 exit code

**問題**：`process.exit(0)` 在 catch block 中會讓 CI 誤認為成功

**正確**：
```typescript
try {
  // ...
} catch (error) {
  console.error(error)
  process.exit(1)  // 錯誤時必須 exit(1)
}
```

## 執行測試的最佳實踐

### 本地開發

1. 確保 dev server 在 port 3001 運行：`PORT=3001 bun dev`
2. 確保測試帳號已建立
3. 執行測試：`bunx playwright test`
4. 查看報告：`bunx playwright show-report`

### CI 環境

1. 設定環境變數（`E2E_ADMIN_EMAIL`, `E2E_ADMIN_PASSWORD`）
2. 啟動 dev server 作為 webServer
3. `bunx playwright test --reporter=github`
4. 上傳 test-results/ 作為 artifact

### 除錯失敗測試

```bash
# 帶 trace 執行
bunx playwright test --trace on

# 執行單一測試
bunx playwright test -g "儲存時 Slug 從 Title 自動生成"

# UI 模式
bunx playwright test --ui

# Debug 模式（step by step）
bunx playwright test --debug
```
