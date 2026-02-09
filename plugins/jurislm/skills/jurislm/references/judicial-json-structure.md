# 司法院開放資料 JSON 結構

司法院開放資料平台 JSON 結構對應說明。051-054 四個分類使用不同的資料結構。

## 結構總覽

| 分類 | 名稱 | JSON 結構 | ID 欄位 | 日期欄位 | 日期格式 |
|------|------|-----------|---------|----------|----------|
| 051 | 裁判書 | 扁平結構 | `JID` | `JDATE` | `YYYYMMDD` |
| 052 | 大法官解釋 | 巢狀 `data` | `data.inte_no` | `data.inte_date` | `YYYY/M/D 上午/下午 HH:MM:SS` |
| 053 | 憲法法庭判決 | 巢狀 `data` | `data.ruling_no` | `data.ruling_date` | `YYYY/M/D 上午/下午 HH:MM:SS` |
| 054 | 憲法法庭裁定 | 巢狀 `data` | `data.ruling_no` | `data.ruling_date` | `YYYY/M/D 上午/下午 HH:MM:SS` |

## 051 裁判書結構

扁平結構，所有欄位在頂層。

```typescript
interface RawDocument051 {
  JID: string;      // 判決識別碼，格式：{法院代碼},{民國年},{案號類型},{案號},{日期}[,{序號}]
  JYEAR: string;    // 民國年（如 "88"）
  JCASE: string;    // 案件類型（如 "台抗"）
  JNO: string;      // 案號（如 "101"）
  JDATE: string;    // 判決日期，格式：YYYYMMDD（如 "19990311"）
  JTITLE: string;   // 判決標題
  JFULL: string;    // 判決全文
  JPDF?: string;    // PDF 連結（可選）
}
```

**範例**：
```json
{
  "JID": "TPSV,88,台抗,101,19990311",
  "JYEAR": "88",
  "JCASE": "台抗",
  "JNO": "101",
  "JDATE": "19990311",
  "JTITLE": "聲請裁定停止強制執行程序",
  "JFULL": "最高法院民事裁定...",
  "JPDF": ""
}
```

## 052 大法官解釋結構

巢狀結構，主要資料在 `data` 物件中。

```typescript
interface RawDocument052 {
  data: {
    inte_no: string;           // 解釋號（如 "608"）
    inte_no_title?: string;    // 解釋號標題
    inte_date: string;         // 解釋日期（如 "2006/1/13 上午 12:00:00"）
    inte_order?: string;       // 發布日期
    inte_issue: string;        // 解釋議題
    inte_desc: string;         // 解釋描述（解釋文）
    inte_reason: string;       // 解釋理由書
    inte_en?: string;          // 英文版解釋議題
    inte_issue_en?: string;    // 英文版議題
    inte_desc_en?: string;     // 英文版解釋描述
    inte_reason_en?: string;   // 英文版理由書
    inte_fact?: string;        // 事實摘要
    inte_kind_1?: string;      // 分類 1
    inte_kind_2?: string;      // 分類 2
    data_url: string;          // 資料來源 URL
    other_doc?: string;        // 其他文件
    other_opinion?: string;    // 其他意見書
  };
  addition?: Record<string, string>;  // 附加文件（連結）
}
```

**範例**：
```json
{
  "data": {
    "inte_no": "608",
    "inte_date": "2006/1/13 上午 12:00:00",
    "inte_issue": "繼承後所領股利應課所得稅之函釋違憲？",
    "inte_desc": "遺產稅之課徵...",
    "inte_reason": "憲法第十九條規定..."
  },
  "addition": {
    "1_聲請書": "/download/..."
  }
}
```

## 053/054 憲法法庭結構

053（判決）與 054（裁定）使用相同結構，差異在於 `ruling_type` 欄位值。

```typescript
interface RawDocument053054 {
  data: {
    ruling_name: string;       // 案件名稱
    ruling_type: string;       // "判決"（053）或 "裁定"（054）
    ruling_kind: string;       // 判決/裁定種類
    ruling_no: string;         // 編號（如 "10"）
    ruling_date: string;       // 日期（如 "2022/6/24 上午 12:00:00"）
    ruling_year: string;       // 民國年
    ruling_word: string;       // 字別（如 "憲判"、"憲裁"）
    ruling_content: string;    // 主文內容
    ruling_reason: string;     // 理由書
    ruling_statement?: string; // 聲明
    case_year?: string;        // 案件年度
    case_word?: string;        // 案件字別（如 "會台"）
    case_no?: string;          // 案件號
    inte_no?: string;          // 相關解釋號（若有）
    inte_issue: string;        // 爭議議題
    claimant: string;          // 聲請人
    main_judge?: string;       // 主筆大法官
    video_final_date?: string; // 影片日期
    data_url: string;          // 資料來源 URL
    apply_date?: string;       // 聲請日期
    pub_date?: string;         // 公告日期
    oral_debate?: string;      // 言詞辯論
    oral_debate_video?: string; // 言詞辯論影片
    pub_news?: string;         // 相關新聞
    about_tag?: string;        // 標籤（JSON 字串）
  };
  addition?: Record<string, string>;  // 附加文件（連結）
}
```

**053 範例（憲法法庭判決）**：
```json
{
  "data": {
    "ruling_name": "警消人員獎懲累積達二大過免職案",
    "ruling_type": "判決",
    "ruling_no": "10",
    "ruling_year": "111",
    "ruling_word": "憲判",
    "ruling_date": "2022/6/24 上午 12:00:00",
    "ruling_content": "一、警察人員人事條例第31條...",
    "ruling_reason": "壹、聲請人陳述要旨...",
    "claimant": "徐國堯"
  },
  "addition": {...}
}
```

**054 範例（憲法法庭裁定）**：
```json
{
  "data": {
    "ruling_type": "裁定",
    "ruling_kind": "實體裁定",
    "ruling_no": "57",
    "ruling_year": "111",
    "ruling_word": "憲裁",
    "ruling_date": "2022/2/25 上午 12:00:00",
    "ruling_content": "司法院釋字第812號解釋之效力...",
    "ruling_reason": "一、按憲法訴訟法...",
    "claimant": "彭雲明"
  },
  "addition": {...}
}
```

## 日期格式差異

| 分類 | 格式 | 範例 | 年份範圍 |
|------|------|------|----------|
| 051 | `YYYYMMDD` | `19990311` | 1990+ |
| 052 | `YYYY/M/D 上午/下午 HH:MM:SS` | `2006/1/13 上午 12:00:00` | 1948+（大法官解釋始於 1948 年）|
| 053/054 | `YYYY/M/D 上午/下午 HH:MM:SS` | `2022/6/24 上午 12:00:00` | 2022+（憲法法庭始於 2022 年）|

## 資料量統計

| 分類 | 預估筆數 | 說明 |
|------|----------|------|
| 051 | ~46,600,000（實測）| 各級法院裁判書 |
| 052 | ~800 | 大法官解釋（1948-2022）|
| 053 | ~300 | 憲法法庭判決（2022+）|
| 054 | ~500 | 憲法法庭裁定（2022+）|

## 欄位提取函式

根據分類提取 ID 和日期欄位的參考實作：

```typescript
function extractFields(
  category: string,
  doc: RawDocument
): { id: string | undefined; date: string | undefined } {
  if (category === "051") {
    const d = doc as RawDocument051;
    return { id: d.JID, date: d.JDATE };
  } else if (category === "052") {
    const d = doc as RawDocument052;
    return { id: d.data?.inte_no, date: d.data?.inte_date };
  } else {
    // 053, 054
    const d = doc as RawDocument053054;
    return { id: d.data?.ruling_no, date: d.data?.ruling_date };
  }
}
```

## 注意事項

1. **052-054 的 `data` 可能為 undefined**：處理時需檢查 `data` 是否存在
2. **日期格式需轉換**：052-054 使用中文「上午/下午」格式，需特殊處理
3. **`addition` 欄位**：包含附加文件連結，鍵值為序號+文件名稱
4. **大法官解釋歷史久遠**：最早可追溯至 1948 年，日期驗證需考慮此範圍
