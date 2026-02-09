---
name: lessons-learned
description: This skill should be used when encountering bugs, debugging issues, writing tests, setting up infrastructure, reviewing PRs, or starting new projects. Activate when the user faces errors, test failures, deployment problems, or asks about best practices. Also activate when patterns like "why is this failing", "how to debug", "test not passing", "deployment issue", or "PR review" appear.
---

# 跨專案開發經驗模式庫

從實際踩坑中提煉的 33+ 個關鍵教訓與改進方案，按主題分類。每個模式包含問題描述、根因分析、正確解法。

---

## A. 診斷與除錯（8 個模式）

### 模式 1：環境 vs 代碼

**原則**：先解決環境問題，再考慮改代碼。

| 症狀 | 正確做法 | 錯誤做法 |
|------|----------|----------|
| Docker 沒運行 | 請用戶啟動 Docker | 改測試配置繞過 |
| 工具找不到 | 檢查代碼用什麼（`unrar` vs `unar`） | 猜測工具名稱 |
| `Cannot find module` | 先 `bun install` | 推給其他分支 |

> 來源：多個專案的反覆踩坑

### 模式 2：過度工程化陷阱

**警訊**：修改 3+ 個配置檔來「繞過」問題。

**正確做法**：停下問「有沒有更直接的解法？」

> 來源：JurisLM 基礎設施配置

### 模式 3：系統性診斷

**適用**：複雜基礎設施問題。

**流程**：建立檢查清單 → 交叉驗證 → Context7 查文檔 → 理解架構再下結論。

> 來源：Coolify + Hetzner 部署問題

### 模式 4：信任用戶反饋

- 用戶說「設定好了」→ 直接檢查結果，不質疑
- 用戶說「不對」→ 重新驗證，不辯解

> 來源：多次用戶互動經驗

### 模式 12：Typecheck 失敗處理流程

**流程**：看錯誤 → `grep` package.json → `ls` node_modules → `bun install` → 再次 typecheck。

**關鍵**：不要直接改型別定義來消除錯誤，先確認依賴安裝正確。

> 來源：多個 Next.js 專案

### 模式 13：ESLint 與官方文檔矛盾

**做法**：Context7 查兩邊文檔，找根本原因。

**案例**：ESLint 報 `useEffect` 依賴問題 → 官方推薦 `useSyncExternalStore` 替代 effect。

> 來源：stock 專案 settings page

### 模式 19：類型重構測試同步

**原則**：修改類型定義後，測試的 mock 和預期值必須反映新邏輯。

- 新增欄位 → mock 要加對應值
- 改變回傳型別 → assertion 要更新
- 刪除欄位 → 移除 mock 中的對應值

> 來源：JurisLM agent 型別重構

### 模式 22：PR Review 處理

**流程**：先 `git log` + `git status` 確認狀態 → 比對 commit → 驗證建議適用性。

**原則**：不盲目採納所有建議，驗證每條建議是否適用於當前代碼。

> 來源：stock PR #12 Copilot Review

---

## B. 測試（12 個模式）

### 模式 8：表層 vs 本質驗證

**原則**：提案前先問「這真的在驗證業務邏輯，還是只在檢查表層風格？」

- 測試字串格式 ≠ 測試計算邏輯
- 測試 CSS class ≠ 測試功能行為

> 來源：多個專案 code review

### 模式 14：Vitest 陷阱

| 陷阱 | 正確 | 錯誤 |
|------|------|------|
| 執行測試 | `bun run test`（vitest） | `bun test`（Bun 內建） |
| 清除 mock | `vi.resetModules()`（重置模組緩存） | `vi.clearAllMocks()`（只清記錄） |

**何時用 `resetModules`**：測試環境變數時，需要完全重置模組緩存。

> 來源：JurisLM + stock 專案

### 模式 15：完成聲明紀律

**聲稱「完成」之前必須全通過**：

```bash
bun run test:run     # 全部通過
bun run typecheck    # 無錯
bun run lint         # 無錯
```

> 來源：核心工作原則

### 模式 26：Playwright + Vitest 共存

**問題 1**：Playwright `use(page)` 被 `react-hooks/rules-of-hooks` 誤判。
- 解法：安裝 `eslint-plugin-playwright` + 關閉 `react-hooks/rules-of-hooks`
- **不要** `globalIgnores` 整個目錄

**問題 2**：Vitest 載入 Playwright spec 導致錯誤。
- 解法：`vitest.config.ts` 必須 exclude：`[...configDefaults.exclude, 'tests/e2e/**']`
- **不可覆蓋預設**（會丟失 `node_modules` 等排除項）

> 來源：stock PR #12

### 模式 27：Playwright E2E 陷阱（JurisLM 特定）

| 陷阱 | 解法 |
|------|------|
| Conv auto-trigger 不觸發 | 需 `?trigger=1` + hasUserMessage + !hasAssistantMessage |
| Mobile webkit 點擊失敗 | `locator.evaluate((el: HTMLElement) => el.click())` 或 `test.skip` |
| Mock assistant message 停用 auto-trigger | 依賴 DB seed data |
| `getByText` strict mode 多個匹配 | `{ exact: true }` 精確匹配 |
| contentEditable `toBeFocused` 不可靠 | 測試功能性（click + type + verify） |

> 來源：JurisLM E2E 測試

### 模式 28：globalIgnores 是逃避不是解法

**問題**：完全 ignore 一個目錄 = 失去所有 lint 保護（formatting、type safety、best practices 全部消失）。

**正確做法**：為目標檔案設定專屬 ESLint override，只關閉真正不適用的規則。

**範例**：E2E 測試檔案
```javascript
{
  files: ['tests/e2e/**/*.ts'],
  plugins: { playwright },
  rules: {
    ...playwright.configs['flat/recommended'].rules,
    'react-hooks/rules-of-hooks': 'off',     // Playwright use() 不是 React hook
    '@next/next/no-html-link-for-pages': 'off', // E2E 不需要 Next.js 規則
  },
}
```

> 來源：stock PR #12 — Copilot 指出 `tests/e2e/**` 不該全域忽略

### 模式 29：useSyncExternalStore snapshot 穩定性

**問題**：`getSnapshot` 每次返回新物件 → `Object.is` 永遠 false → 無限 re-render → `Maximum update depth exceeded`。

**根因**：每次 `JSON.parse` 產生新物件引用。

**解法**：cache raw string + parsed result，只在 raw string 改變時才創建新物件。

```typescript
let cachedRaw = '';
let cachedResult = defaultSettings;

function readSettings(): Settings {
  const raw = localStorage.getItem('settings') || '';
  if (raw !== cachedRaw) {
    cachedRaw = raw;
    cachedResult = raw ? JSON.parse(raw) : defaultSettings;
  }
  return cachedResult; // 穩定引用
}
```

> 來源：stock 專案 settings page

### 模式 30：Mock 資料格式必須與 API 回傳型別一致

**問題**：mock 回傳格式與 API 不同 → `undefined.name` → runtime crash。

**案例**：`/api/market/indices` 回傳 `{ taiex: {...}, otc: {...} }`（物件），mock 回傳 `[{...}]`（陣列）。

**最佳實踐**：寫 mock 前先讀 API route handler 的回傳型別 → 從 TypeScript 型別定義反推 mock 結構。

> 來源：stock E2E 測試

### 模式 31：PR Review 修正必須逐項讀檔驗證

**問題**：聲稱「全部修完」但漏了 1 項 = 信任崩壞。

**做法**：修完 N 項後必須逐一 `Read` 對應檔案行號確認實際變更。

**追蹤表格**：
```
| # | 檔案:行號 | 建議 | 狀態 | 驗證 |
|---|-----------|------|------|------|
| 1 | test-base.ts:15 | 改用 **/api/** | ✅ 已修 | ✅ 已讀 |
```

> 來源：stock PR #12 review fixes

### 模式 32：Playwright assertion 陷阱

| 陷阱 | 問題 | 正確做法 |
|------|------|----------|
| `.catch(() => {})` | 吞掉斷言失敗，測試永遠不會 fail | 移除 `.catch()` |
| `not.toBeVisible()` | 語義不清 | 用 `toBeHidden()` 更語義化 |
| 缺少 assertion | 測試無斷言 = 永遠通過 | 每個 test 至少一個 assertion |
| 忘記 `await` | async matcher 未等待 | `toHaveClass()` 等必須 `await` |

**ESLint 規則**：
- `playwright/expect-expect` — 每個 test 至少一個 assertion
- `playwright/missing-playwright-await` — async matcher 必須 await
- `playwright/no-useless-not` — 用 `toBeHidden()` 替代 `not.toBeVisible()`

> 來源：stock PR #12 eslint-plugin-playwright

### 模式 33：Merge + Review fixes 分開 commit

**問題**：merge commit 包含非 merge 的修改 → git history 混亂。

**策略**：
1. `git stash` review fixes
2. 完成 merge commit
3. `git stash pop`
4. 建立 review fixes commit

> 來源：stock PR #12 merge + review 流程

---

## C. 基礎設施與部署（7 個模式）

### 模式 5：MCP Server 重複配置

**問題**：同一服務有獨立安裝和 Plugin 兩套工具。

**診斷**：`env | grep TOKEN` + `ls ~/.claude/plugins/`

**解法**：移除獨立安裝（`.claude.json` 中的 `mcpServers`），統一使用 Plugin。

> 來源：jurislm-plugins MCP 遷移

### 模式 6：MCP 參數不匹配

**做法**：Context7 查 API 文檔確認參數。

**備案**：用 `curl` 直接呼叫 API 繞過 MCP。

> 來源：Coolify MCP Server

### 模式 7：操作前檢查完整狀態

**原則**：建立前先列出（DNS records、applications），注意 wildcard。

**案例**：建 subdomain 前先檢查 DNS wildcard 是否已涵蓋。

> 來源：Cloudflare + Coolify 部署

### 模式 16：Coolify Service vs Application

| 類型 | 更新 FQDN | 可更新欄位 |
|------|-----------|-----------|
| Application | 直接更新 | FQDN + 其他設定 |
| Service | 修改 `docker_compose_raw` 的 Traefik labels | name, description, docker_compose_raw |

> 來源：Coolify 部署管理

### 模式 17：Hetzner Volume 與資料遷移

**容量估算**：Embedding ≈ 維度 × 4B × 記錄數。

**PostgreSQL 遷移流程**：停容器 → 複製資料（`chown 999:999`）→ symlink → 啟動。

> 來源：jurislm-shared-db Volume 遷移

### 模式 18：大量資料處理

- `UNNEST` + `ON CONFLICT` 批次 10K 筆
- 每單位 checkpoint（非 phase）
- 背景監控查 DB 最可靠

> 來源：JurisLM 資料同步

### 模式 21：Release Please

**版本規則**：
- `feat:` → MINOR（0.1.0 → 0.2.0）
- `fix:` → PATCH（0.1.0 → 0.1.1）
- `feat!:` / `BREAKING CHANGE:` → MAJOR

**前置需求**：GitHub Repo Settings → Actions → 允許 GitHub Actions 建立 PR。

> 來源：stock + lawyer-app Release Please 配置

### 模式 23：Migration 安全

**流程**：備份欄位 → 備份資料 → 刪舊 constraint → 轉換 → 驗證（0 筆異常）→ 新 constraint。

> 來源：JurisLM 資料庫遷移

---

## D. 安全與錯誤處理（4 個模式）

### Finally block 必須用 nested try-catch

**問題**：`finally { await cleanup1(); await cleanup2(); controller.close(); }` → cleanup1 拋錯會跳過後續。

**正確做法**：每個 async 操作獨立 try-catch，確保 `controller.close()` 一定執行。

```typescript
finally {
  try { await cleanup1(); } catch (e) { /* log */ }
  try { await cleanup2(); } catch (e) { /* log */ }
  controller.close(); // 一定執行
}
```

> 來源：JurisLM PR #134 — SSE stream memory leak

### Tool 執行需要 timeout 保護

- `Promise.race` + `setTimeout` 包裝所有外部工具呼叫
- 避免單一工具掛住整個 agentic loop
- 預設 30 秒，可配置

> 來源：JurisLM agent tool execution

### Per-resource rate limiting

**問題**：全域 rate limit 不夠 → 單一資源被 rapid-fire 攻擊。

**解法**：需要 per-conversation / per-resource 限制。

> 來源：JurisLM API 安全

### Sanitize 策略：替換優於刪除

- 危險 pattern 替換為 `[REDACTED]`（而非靜默移除）
- 保留 context 便於安全審計
- 正規化空白時保留換行符（`[^\S\n]+` 而非 `\s+`）

> 來源：JurisLM content sanitization

---

## E. 架構與重構（2 個模式）

### Config-driven mapping 優於 switch

**問題**：switch 隨 case 增長難以維護。

**解法**：`Record<string, Type>` map + fallback 比 switch 更易擴展。

**案例**：`TOOL_TO_AGENT_TYPE` map 取代 `detectAgentType` switch。

> 來源：JurisLM agent routing

### Token budget 控制

- Agentic loop 必須有 token 總預算上限
- 超過時 graceful 中斷，不要無聲耗盡 context
- 追蹤累積 usage 並在每輪檢查

> 來源：JurisLM agent conversation

---

## F. Git 工作流（2 個模式）

### Merge + Review fixes 分開 commit

見模式 33。

### PR Review 驗證流程

1. 收到 review comments
2. 逐項修正（用表格追蹤）
3. 每項修正後 `Read` 對應檔案行號驗證
4. 全部修正完成 → 跑完整驗證指令
5. commit + push

> 來源：stock PR #12

---

## G. 工具與工作流（2 個模式）

### OpenSpec 結構化變更

**4 個 artifacts**：proposal → design → specs（GIVEN/WHEN/THEN）→ tasks。

**用途**：大型功能開發的結構化流程。

> 來源：stock + JurisLM 功能開發

### Task tool subagent_type

**做法**：用完整名稱（如 `code-simplifier:code-simplifier`），查 `Available agents` 列表。

> 來源：Claude Code 使用經驗

---

## D2. 業務邏輯（4 個模式）

### JurisLM 核心業務

- 三層知識庫：無綁定 → 共享+個人；有綁定 → 三層
- 使用者/專案隔離
- Unified Agent（SKILL.md + 11 Tools）
- Taxonomy 同義詞擴展

### Legal Plugin 設計模式

- GREEN/YELLOW/RED 分類 + 4 種升級觸發器
- Playbook（MD + YAML frontmatter）

### 分類系統

- Contract/NDA = 3 級 GREEN/YELLOW/RED
- Risk = 4 級 + ORANGE

### Prompt Injection 防護

**流程**：`sanitizeContent()` → 結構化 JSON → Zod 驗證 → 不執行 LLM 回應的系統操作。

> 來源：JurisLM security review
