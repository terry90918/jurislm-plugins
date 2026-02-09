# Stock API Reference

完整的 API endpoints 文件。所有 endpoints 統一回傳 `{ success: boolean, data?: T, error?: string }`。

## Portfolio 管理

| Method | Path | 說明 |
|--------|------|------|
| GET | `/api/portfolio` | 取得預設投資組合（含 holdings） |
| POST | `/api/portfolio/transactions` | 新增交易（BUY/SELL） |
| DELETE | `/api/portfolio/transactions/[id]` | 刪除交易 |
| GET | `/api/portfolio/[id]/news` | 取得投資組合相關新聞 |
| GET | `/api/portfolio/benchmark` | 投資組合 vs 0050 對標分析 |
| GET | `/api/portfolio/benchmark/yearly` | 逐年績效比較 |
| GET | `/api/portfolio/dividend-summary` | 股息摘要與預測 |
| GET | `/api/portfolio/quarterly` | 季度價格區間分析 |
| GET | `/api/portfolio/history` | 投資組合歷史價值 |

## 個股資料

| Method | Path | 說明 |
|--------|------|------|
| GET | `/api/stock/[symbol]` | 即時報價 |
| GET | `/api/stock/[symbol]/profile` | 公司基本資料 |
| GET | `/api/stock/[symbol]/history` | 歷史價格 |
| GET | `/api/stock/[symbol]/news` | 個股新聞 |
| GET | `/api/stock/[symbol]/financials` | 財務報表（損益、資產負債、現金流量） |
| GET | `/api/stock/[symbol]/quarterly` | 季度價格區間 |
| GET | `/api/stock/[symbol]/related` | 相關個股 |
| GET | `/api/stock/search` | 搜尋股票（symbol/name） |
| GET | `/api/search/suggestions` | 搜尋建議 |
| GET/POST | `/api/stock-price` | 報價快取讀取/寫入 |

## 觀察清單

| Method | Path | 說明 |
|--------|------|------|
| GET | `/api/watchlist` | 列出觀察清單 |
| POST | `/api/watchlist` | 加入觀察清單 |
| DELETE | `/api/watchlist/[symbol]` | 從觀察清單移除 |
| PATCH | `/api/watchlist/[symbol]/move` | 移動至群組 |
| GET | `/api/watchlist/groups` | 列出群組 |
| POST | `/api/watchlist/groups` | 建立群組 |
| GET | `/api/watchlist/groups/[id]` | 群組詳情 |
| PATCH | `/api/watchlist/groups/[id]` | 更新群組 |
| DELETE | `/api/watchlist/groups/[id]` | 刪除群組 |
| POST | `/api/watchlist/groups/reorder` | 群組排序 |
| GET | `/api/watchlist/groups/[id]/overlap` | ETF 重疊分析 |

## 市場資料

| Method | Path | 說明 |
|--------|------|------|
| GET | `/api/market/indices` | 市場指數（加權/櫃買/電子/金融） |
| GET | `/api/market/rankings` | 熱門排行（成交量/漲幅/跌幅） |
| GET | `/api/market/etf` | ETF 排行 |
| GET | `/api/market/intraday` | 盤中走勢 |
| GET | `/api/market/futures` | 期貨報價 |

## ETF 資料

| Method | Path | 說明 |
|--------|------|------|
| GET | `/api/etf/screen` | ETF 篩選（theme/region） |
| GET | `/api/etf/compare` | ETF 比較 |
| GET | `/api/etf/[symbol]/holdings` | ETF 成分股 |
| GET | `/api/etf/news` | ETF 新聞 |
| GET | `/api/etf/trending` | 熱門 ETF |
| GET | `/api/etf/articles` | ETF 教學文章 |
| GET | `/api/etf/newly-listed` | 新上市 ETF |
| GET | `/api/etf/themes` | ETF 題材與績效 |
| GET | `/api/etf/recommended` | 推薦 ETF |

## 國際市場

| Method | Path | 說明 |
|--------|------|------|
| GET | `/api/international/indices` | 國際指數（S&P500/Nasdaq/DJI） |
| GET | `/api/international/forex` | 外匯報價 |
| GET | `/api/international/us-stocks` | 美股資料 |
| GET | `/api/international/adr` | 台灣 ADR |
| GET | `/api/international/hk-stocks` | 港股 |
| GET | `/api/international/commodities` | 大宗商品 |
| GET | `/api/international/crypto` | 加密貨幣 |

## 新聞

| Method | Path | 說明 |
|--------|------|------|
| GET | `/api/news/featured` | 精選新聞 |
| GET | `/api/news/taiwan` | 台股新聞 |
| GET | `/api/news/us` | 美股新聞 |

## 股息與日曆

| Method | Path | 說明 |
|--------|------|------|
| POST | `/api/dividend/save` | 儲存股息資料 |
| POST | `/api/dividend/sync` | 同步 Yahoo Finance 股息 |
| GET | `/api/calendar/events` | 行事曆事件 |
| GET | `/api/calendar/earnings` | 財報公告日 |

## 回傳格式範例

### 成功回傳

```json
{
  "success": true,
  "data": {
    "symbol": "2330",
    "name": "台積電",
    "price": 600,
    "change": 10,
    "changePercent": 1.69
  }
}
```

### 錯誤回傳

```json
{
  "success": false,
  "error": "找不到該股票資料"
}
```

### Market Indices 回傳（注意：物件，非陣列）

```json
{
  "success": true,
  "data": {
    "taiex": { "code": "TAIEX", "name": "加權指數", "value": 22500.3, "change": 120.5, "changePercent": 0.54 },
    "otc": { "code": "OTC", "name": "櫃買指數", "value": 250.8, "change": 2.1, "changePercent": 0.84 },
    "electronics": { "code": "ELECTRONICS", "name": "電子類指數", "value": 1150.2, "change": 15.3, "changePercent": 1.35 },
    "financial": { "code": "FINANCIAL", "name": "金融保險指數", "value": 1980.5, "change": -5.2, "changePercent": -0.26 },
    "updatedAt": "2024-06-01T06:00:00Z"
  }
}
```
