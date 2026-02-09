---
name: stock
description: This skill should be used when the user asks to develop features for the "台股看盤" stock app, work with TWSE/Yahoo Finance APIs, manage portfolio or watchlist functionality, build ETF analysis features, or debug stock-related components. Activate when user mentions stock.jurislm.com, terry90918/stock repo, or Taiwan stock market app development.
---

# 台股看盤應用開發指南

台股看盤是一個 Taiwan stock portfolio tracking & analysis 應用，提供即時 TWSE/OTC 報價、投資組合管理、股息分析、ETF 重疊分析與 0050 績效對標。

## 架構概覽

```
TWSE/Yahoo API → src/lib/ (fetch + 商業邏輯) → src/app/api/ (路由) → src/hooks/ (客戶端狀態) → src/components/ (UI)
```

### 資料流模式

| 層 | 職責 | 位置 |
|----|------|------|
| 外部 API | TWSE OpenAPI、Yahoo Finance、RSS | 網路請求 |
| Lib | 純邏輯 + 快取（無 React） | `src/lib/` |
| API Routes | JSON endpoints，統一回傳格式 | `src/app/api/` |
| Hooks | 客戶端狀態管理 + TTL 快取 | `src/hooks/` |
| Components | Server/Client 元件分離 | `src/components/` |
| Database | PostgreSQL + Prisma ORM | `prisma/schema.prisma` |

### API 回傳格式

所有 API endpoints 統一回傳：

```ts
{ success: boolean, data?: T, error?: string }
```

Dynamic route params 使用 Next.js 15+ 的 `params: Promise<{ param: string }>` pattern（必須 await）。

## 核心模組

### 資料庫 Schema

| Model | 用途 | 關鍵關聯 |
|-------|------|----------|
| Portfolio | 投資組合容器 | → holdings[], watchlistGroups[] |
| Holding | 觀察清單/持股項目 | → portfolio, group?, transactions[] |
| Transaction | 買賣紀錄（BUY/SELL） | → holding（cascade delete） |
| WatchlistGroup | 群組管理 | → portfolio（cascade）, holdings[] |
| Dividend | 除權息資料 | unique: [symbol, year, quarter] |
| StockPrice | 報價快取 | unique: symbol |

**重要限制**：
- `WATCHLIST_LIMIT = 50`（每組合觀察清單上限）
- `GROUP_LIMIT = 20`（群組數量上限）
- `GROUP_NAME_MAX_LENGTH = 20`（群組名稱長度上限）
- 股票代號格式：TWSE 格式（如 `"2330"`），Yahoo Finance 查詢附加 `.TW` / `.TWO`

### Lib 模組重點

| 模組 | 功能 | 快取策略 |
|------|------|----------|
| `twse.ts` | TWSE API 整合 | 股票列表 1 小時，日報價 5 分鐘 |
| `yahoo-finance.ts` | 歷史價格 + 股息 | 無快取（按需） |
| `stock-price.ts` | 報價 DB 快取層 | 寫入 DB，讀取時 fallback |
| `benchmark.ts` | 投資組合 vs 0050 對標 | — |
| `benchmark-yearly.ts` | 逐年績效比較 | — |
| `portfolio-calc.ts` | 交易處理（加權平均成本法） | — |
| `quarterly-analysis.ts` | 季度價格區間熱圖 | — |
| `etf-overlap.ts` | ETF 重疊分析（Jaccard 係數） | — |
| `formatters.ts` | 數字格式化工具 | — |

### Hook 模式

所有 custom hook 遵循相同 pattern：

```ts
function useXxx() {
  const [data, setData] = useState<T>(null);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  const fetchData = useCallback(async () => { /* fetch /api/... */ }, []);
  useEffect(() => { fetchData(); }, [fetchData]);

  return { data, isLoading, error, refresh: fetchData, ...actions };
}
```

**關鍵 hook**：
- `usePortfolio()` — 最複雜，先 fetch portfolio 再用 `Promise.all` 載入所有報價
- `useMarketIndices()` — data 是 `MarketOverview` 物件（`{taiex, otc, electronics, financial}`），**不是陣列**
- `useBenchmark()` / `useYearlyBenchmark()` — 5 分鐘客戶端 TTL 快取
- `useHotRankings()` — 5 分鐘快取，支援切換 volume/gainers/losers

### 元件命名規則

- Domain-prefixed：`StockCard`、`HoldingCard`、`PortfolioSummary`
- Server/Client 分離：`XxxDetail.tsx`（server）+ `XxxDetailClient.tsx`（client）
- Barrel export：`src/components/index.ts`

## 常見開發工作流

### 新增功能（Feature Development）

遵循 schema → lib → API → hook → component → page 順序：

1. 修改 `prisma/schema.prisma` → `bunx prisma db push`
2. 建立 `src/lib/xxx.ts`（純邏輯 + 測試）
3. 建立 `src/app/api/xxx/route.ts`（統一回傳格式）
4. 建立 `src/hooks/useXxx.ts`（遵循 hook pattern）
5. 建立 `src/components/xxx/`（Server/Client 分離）
6. 整合至頁面

### 新增 API Endpoint

```ts
// src/app/api/xxx/route.ts
import { NextResponse } from 'next/server';

export async function GET() {
  try {
    const data = await fetchSomething();
    return NextResponse.json({ success: true, data });
  } catch (error) {
    return NextResponse.json(
      { success: false, error: '取得資料失敗' },
      { status: 500 }
    );
  }
}
```

### 偵錯報價問題

1. 確認 TWSE API 是否可用（開盤時間：09:00-13:30 台灣時間）
2. 檢查 `stock-price.ts` 的 DB 快取是否過期
3. Yahoo Finance 附加 `.TW`（上市）或 `.TWO`（上櫃）後綴
4. 使用 `getCachedStockPrice()` 作為 fallback

## 測試策略

### 單元測試（Vitest）

- 1847 tests，coverage ≥ 95%
- 測試檔案：`*.test.ts` 或 `__tests__/` 子目錄
- Mocking：`vi.fn()` for fetch，`renderHook`/`waitFor`/`act` for hooks
- 執行：`bun run test:run`（非 `bun test`）

### E2E 測試（Playwright）

- 66 tests across 12 spec files
- Auto mock fixture：`tests/e2e/fixtures/test-base.ts`（攔截 `**/api/**`）
- Mock data：`tests/e2e/fixtures/mock-data.ts`
- 全局預熱：`tests/e2e/global-setup.ts`（避免首次編譯超時）
- ESLint：使用 `eslint-plugin-playwright` 專屬規則
- 未 mock 的 API 回傳 404（hermetic tests）
- 執行：`bunx playwright test`

### 驗證指令

```bash
bun run typecheck && bun run lint && bun run test:run  # 必須全通過
bunx playwright test                                     # E2E 驗證
```

## 已知陷阱

### 資料庫操作

- **count-then-create 必須用 `$transaction`**：先 count 檢查限制再 create 有 race condition，必須包在 `prisma.$transaction()` 內
- **Reorder API 需驗證 ID 陣列**：排序 endpoint 接收 `orderedIds` 時，必須驗證無重複、全部存在、完整集合（不可遺漏）
- **重構實作後同步更新測試 mock**：將 raw Prisma 呼叫改為 hook method 時，測試的 mock target 也要跟著改

### 導航

- **SideMenu 連結需帶完整 query params**：側邊選單的頁面連結（如 `/portfolio`）需保留當前的 query parameters（如 `?tab=holdings`），否則導航會丟失 tab 狀態

### React 狀態

- `useSyncExternalStore` 的 `getSnapshot` 必須返回 **referentially stable** 的值
- 每次返回新物件 → `Object.is` 永遠 false → 無限 re-render
- Settings page 使用 `cachedSettings` + `cachedSettingsRaw` 解決

### Mock 資料

- `/api/market/indices` 回傳 `MarketOverview` 物件（`{taiex, otc, ...}`），**不是陣列**
- 寫 mock 前先讀 API route handler 的回傳型別
- E2E mock 的 `page.route()` 用 `**/api/**`（非 `/api/**`）

### ESLint

- E2E 檔案用 `eslint-plugin-playwright`，不要 `globalIgnores`
- Vitest exclude 必須延伸：`[...configDefaults.exclude, 'tests/e2e/**']`

### Playwright

- `expect(x).toBeVisible().catch(() => {})` 會吞掉斷言失敗
- `not.toBeVisible()` → 用 `toBeHidden()`
- `toHaveClass()` 等 async matcher 必須 `await`

## 附加資源

### 參考文件

- **`references/api-reference.md`** — 完整 API endpoints 文件
- **`references/architecture.md`** — 詳細架構與資料流

### 外部連結

- [Stock App](https://stock.jurislm.com)
- [GitHub Repo](https://github.com/terry90918/stock)
- [TWSE OpenAPI](https://openapi.twse.com.tw/)
- [Yahoo Finance API (yahoo-finance2)](https://github.com/gadicc/node-yahoo-finance2)
