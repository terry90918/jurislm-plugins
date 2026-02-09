# 法規 JSON 結構文件

法務部全國法規資料庫開放 API 官方 JSON 結構對應說明。

## 官方資料來源

### Open API 端點

| 端點 | 說明 | 格式 |
|------|------|------|
| `https://law.moj.gov.tw/api/Ch/Law/JSON` | 中文法律 | JSON/XML |
| `https://law.moj.gov.tw/api/Ch/Order/JSON` | 中文命令 | JSON/XML |
| `https://law.moj.gov.tw/api/En/Law/JSON` | 英文法律 | JSON/XML |
| `https://law.moj.gov.tw/api/En/Order/JSON` | 英文命令 | JSON/XML |

### 下載檔案結構

下載的 ZIP 檔案包含：

| 檔案 | 說明 |
|------|------|
| `ChLaw.json` | 法規資料（約 25MB，1,340+ 筆法律）|
| `schema.csv` | 欄位定義 |
| `manifest.csv` | 資料清單 |

**注意**：JSON 檔案使用 UTF-8 BOM 編碼，讀取時須使用 `encoding='utf-8-sig'`

## JSON 結構

### 頂層結構

```json
{
  "UpdateDate": "2025/12/19 上午 12:00:00",
  "Laws": [
    { /* 法規物件 */ }
  ]
}
```

### 法規物件欄位

| JSON 欄位 | 中文名稱 | 資料類型 | 說明 |
|-----------|----------|----------|------|
| `LawLevel` | 法規位階 | string | 憲法、法律、命令等 |
| `LawName` | 法規名稱 | string | 法規全名 |
| `LawURL` | 法規網址 | string | 法規資料庫連結，含 pcode 參數 |
| `LawCategory` | 法規類別 | string | 分類路徑（如「行政/法務部」）|
| `LawModifiedDate` | 法規異動日期 | string | 格式 YYYYMMDD |
| `LawEffectiveDate` | 生效日期 | string | 格式 YYYYMMDD，可為空 |
| `LawEffectiveNote` | 生效內容 | string | 生效說明，可為空 |
| `LawAbandonNote` | 廢止註記 | string | 廢止說明，可為空 |
| `LawHasEngVersion` | 是否英譯註記 | string | Y/N |
| `EngLawName` | 英文法規名稱 | string | 英文名稱，可為空 |
| `LawAttachements` | 附件 | array | 附件檔案清單 |
| `LawHistories` | 沿革內容 | string | 立法歷程（多行文字）|
| `LawForeword` | 前言 | string | 法規前言，約 0.05% 法規有此欄位 |
| `LawArticles` | 法條 | array | 條文清單 |

### LawArticles 條文結構

```json
{
  "ArticleType": "A",
  "ArticleNo": "第 1 條",
  "ArticleContent": "條文內容..."
}
```

| 欄位 | 說明 |
|------|------|
| `ArticleType` | C = 章節標題，A = 條文 |
| `ArticleNo` | 條號（章節時為空）|
| `ArticleContent` | 條文或章節內容 |

### LawAttachements 附件結構

```json
{
  "FileName": "附件一.pdf",
  "FileURL": "https://law.moj.gov.tw/..."
}
```

| 欄位 | 說明 |
|------|------|
| `FileName` | 附件檔案名稱 |
| `FileURL` | 附件下載網址 |

## pcode 提取

從 `LawURL` 提取 pcode（法規唯一識別碼）：

```
LawURL: https://law.moj.gov.tw/LawClass/LawAll.aspx?pcode=A0000001
pcode: A0000001
```

提取方式：解析 URL query string 的 `pcode` 參數

## JSON 範例

```json
{
  "UpdateDate": "2025/12/19 上午 12:00:00",
  "Laws": [
    {
      "LawLevel": "憲法",
      "LawName": "中華民國憲法",
      "LawURL": "https://law.moj.gov.tw/LawClass/LawAll.aspx?pcode=A0000001",
      "LawCategory": "憲法",
      "LawModifiedDate": "19470101",
      "LawEffectiveDate": "",
      "LawEffectiveNote": "",
      "LawAbandonNote": "",
      "LawHasEngVersion": "Y",
      "EngLawName": "Constitution of the Republic of China (Taiwan)",
      "LawAttachements": [],
      "LawHistories": "中華民國三十五年十二月二十五日...",
      "LawForeword": "中華民國國民大會受全體國民之付託...",
      "LawArticles": [
        {
          "ArticleType": "C",
          "ArticleNo": "",
          "ArticleContent": "第 一 章 總綱"
        },
        {
          "ArticleType": "A",
          "ArticleNo": "第 1 條",
          "ArticleContent": "中華民國基於三民主義，為民有民治民享之民主共和國。"
        }
      ]
    }
  ]
}
```

## 資料庫對應

### laws 資料表對應

| JSON 欄位 | 資料庫欄位 | 型別 | 備註 |
|-----------|------------|------|------|
| `LawURL` (pcode) | `pcode` | VARCHAR(20) | 主鍵，從 URL 提取 |
| `LawName` | `law_name` | VARCHAR(500) | |
| `LawLevel` | `law_level` | VARCHAR(20) | 憲法/法律/命令 |
| `LawCategory` | `law_category` | VARCHAR(100) | 同時透過 law_category_mappings 關聯 |
| `LawModifiedDate` | `law_modified_date` | VARCHAR(8) | 格式 YYYYMMDD |
| `LawURL` | `law_url` | TEXT | 完整 URL |
| `LawEffectiveDate` | `effective_date` | VARCHAR(8) | 格式 YYYYMMDD |
| `LawEffectiveNote` | `effective_note` | TEXT | |
| `LawAbandonNote` | `abolish_note` | TEXT | |
| `LawHasEngVersion` | `has_english` | BOOLEAN | Y -> true, N -> false |
| `EngLawName` | `english_name` | VARCHAR(500) | |
| `LawHistories` | `law_histories` | TEXT | |
| `LawArticles` | `law_articles` | JSONB | 保留原始結構 |

### law_forewords 資料表對應

| JSON 欄位 | 資料庫欄位 | 型別 | 備註 |
|-----------|------------|------|------|
| `LawForeword` | `foreword_content` | TEXT | |
| - | `law_id` | INTEGER | 外鍵關聯 laws(law_id) |

### law_attachments 資料表對應

| JSON 欄位 | 資料庫欄位 | 型別 | 備註 |
|-----------|------------|------|------|
| `FileName` | `file_name` | VARCHAR(500) | |
| `FileURL` | `file_url` | TEXT | |
| - | `law_id` | INTEGER | 外鍵關聯 laws(law_id) |

### law_categories 與 law_category_mappings

`LawCategory` 欄位（如「行政/法務部/法制目」）需解析為樹狀結構：

1. 解析分類路徑
2. 建立或查詢 law_categories 記錄
3. 建立 law_category_mappings 關聯

## 技術注意事項

### 編碼處理

```python
import json

# 必須使用 utf-8-sig 處理 BOM
with open("ChLaw.json", encoding="utf-8-sig") as f:
    data = json.load(f)
```

### 日期格式轉換

```python
from datetime import datetime

def parse_date(date_str: str) -> datetime | None:
    if not date_str:
        return None
    return datetime.strptime(date_str, "%Y%m%d")
```

### pcode 提取

```python
from urllib.parse import urlparse, parse_qs

def extract_pcode(law_url: str) -> str:
    parsed = urlparse(law_url)
    params = parse_qs(parsed.query)
    return params.get("pcode", [""])[0]
```

## 相關資源

- 法務部全國法規資料庫：https://law.moj.gov.tw/
- Open API 首頁：https://law.moj.gov.tw/api/
- 資料流程圖：`jurislm_db/docs/data-flow.md`
