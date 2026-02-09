# Stock 架構文件

## 技術棧

| 類別 | 技術 | 版本 |
|------|------|------|
| 框架 | Next.js (App Router) | 16.x |
| UI | React | 19.x |
| 資料庫 | PostgreSQL + Prisma ORM | 6.x |
| 圖表 | lightweight-charts | 5.x |
| 力導向圖 | d3-force + react-force-graph-2d | — |
| 外部 API | yahoo-finance2 | 3.x |
| 樣式 | Tailwind CSS | 4 |
| 測試 | Vitest + RTL / Playwright | — |
| 套件管理 | Bun | — |

## 頁面路由

| Path | 說明 | 元件模式 |
|------|------|----------|
| `/` | 首頁（雙欄佈局，breakpoint 840px） | Server + Client |
| `/portfolio` | 投資組合管理（觀察清單/持股 Tab） | Client Heavy |
| `/stock/[symbol]` | 個股詳情 | Server → Client |
| `/markets` | 市場趨勢排行 | Server + Client |
| `/etf` | ETF 總覽 | Server + Client |
| `/etf/screen` | ETF 篩選 | Client Heavy |
| `/etf/compare` | ETF 比較 | Client Heavy |
| `/international` | 國際市場 | Server + Client |
| `/settings` | 設定 | Client |

## 資料庫關聯

```
Portfolio (1)
  ├──→ (N) Holding ←── unique: [portfolioId, symbol, groupId]
  │         ├──→ (N) Transaction (cascade delete)
  │         └──→ (0..1) WatchlistGroup
  └──→ (N) WatchlistGroup (cascade delete)
              └──→ (N) Holding (set null on delete)

Dividend (standalone) ←── unique: [symbol, year, quarter]
StockPrice (standalone) ←── unique: symbol
```

### Cascade 行為

- 刪除 Portfolio → 刪除所有 Holdings 和 WatchlistGroups
- 刪除 Holding → 刪除所有 Transactions
- 刪除 WatchlistGroup → Holding.groupId 設為 null（不刪除 Holding）

## 資料流詳解

### 1. 報價流程

```
用戶瀏覽個股
  → Server Component 載入頁面
  → Client hook usePortfolio() / useStockQuote()
  → fetch /api/stock/[symbol]
  → lib/twse.ts fetchStockQuote()
  → TWSE API（開盤時間 09:00-13:30）
  → saveStockPrice() 寫入 DB 快取
  → 回傳報價

  fallback（API 失敗時）:
  → getCachedStockPrice() 從 DB 讀取
  → 附加 isCached: true 標記
```

### 2. 投資組合計算流程

```
Portfolio 頁面載入
  → usePortfolio() hook
  → fetch /api/portfolio
  → getOrCreatePortfolio() 取得/建立預設組合
  → 每個 Holding 透過 calculateHoldingStats() 計算：
     - 加權平均成本法（Average Cost Method）
     - totalShares, avgCost, totalCost, realizedGain
  → Client 端用 Promise.all 並行載入所有報價
  → 計算 currentValue, profitLoss（未實現損益）
  → PortfolioSummary 彙總全部持股
```

### 3. 0050 對標流程

```
/api/portfolio/benchmark
  → 從 benchmark-data.ts 取得 0050 成分股靜態資料
  → BASELINE_DATE = 2003-07-01（0050 成立日）
  → pre-2008 價格：benchmark-0050-history.ts（靜態資料）
  → post-2008 價格：Yahoo Finance API
  → 比較投資組合 vs 同期買 0050 的績效差異

/api/portfolio/benchmark/yearly
  → 逐年比較（含股息再投資）
```

### 4. ETF 重疊分析

```
/api/watchlist/groups/[id]/overlap
  → etf-overlap.ts
  → 取得群組內所有 ETF 的成分股
  → 計算 Jaccard Index（by count）: |A ∩ B| / |A ∪ B|
  → 計算 Overlap by weight: Σ min(weight_A, weight_B)
  → 視覺化：react-force-graph-2d + D3 force simulation
```

## 快取策略

| 層級 | 位置 | TTL | 說明 |
|------|------|-----|------|
| TWSE 股票列表 | Memory | 1 小時 | `twse.ts` 模組層級 |
| TWSE 日報價 | Memory | 5 分鐘 | `twse.ts` 模組層級 |
| 報價 DB 快取 | PostgreSQL | 永久 | `stock-price.ts`，API 失敗時 fallback |
| Benchmark 分析 | Client | 5 分鐘 | `useBenchmark()` hook |
| 排行資料 | Client | 5 分鐘 | `useHotRankings()` hook |
| 市場指數 | Client | 1 分鐘 | `useMarketIndices()` hook |

## Server vs Client Components

### 模式：Server Component 包裝 Client Component

```
StockQuoteDetail.tsx (Server)
  → 取得初始資料（SEO friendly）
  → 傳給 StockQuoteDetailClient.tsx (Client)
     → 使用 hooks 取得即時更新
     → 處理互動（按鈕、表單）
```

### 使用 `'use client'` 的時機

- 需要 hooks（useState, useEffect, useCallback）
- 需要事件處理（onClick, onChange）
- 需要瀏覽器 API（localStorage, window）

## 首頁雙欄佈局

Google Finance 風格，breakpoint 840px（`min-[840px]:`）：

**左欄**：WatchlistTopMovers → WatchlistSection → TodayNews → RecentSearches

**右欄（sticky）**：PortfolioSummary → MarketTrends → EarningsCalendar

## 環境變數

| 變數 | 說明 |
|------|------|
| `DATABASE_URL` | PostgreSQL 連線字串 |

**Production**：Coolify 環境變數（stock.jurislm.com）
**Database**：Port 5445（Hetzner 伺服器 46.225.58.202）
