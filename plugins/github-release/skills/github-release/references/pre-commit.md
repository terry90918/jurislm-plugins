# Git Hook: Husky Pre-commit

## 檔案位置

`.husky/pre-commit`

## 用途

在每次 `git commit` 前自動執行 5 項檢查，確保代碼品質。任一項失敗則阻止 commit。

## 執行順序

| 步驟 | 指令 | 說明 |
|------|------|------|
| 1. format | `bun format` | Prettier 自動格式化 + 重新暫存 |
| 2. lint | `bun lint` | ESLint 代碼風格檢查 |
| 3. typecheck | `bun typecheck` | TypeScript 型別安全驗證 |
| 4. test | `bunx vitest run` | Vitest 單元測試 |
| 5. openspec | `bunx @fission-ai/openspec validate --specs` | OpenSpec 規格文件驗證 |

## 完整內容

```sh
#!/bin/sh
#
# Pre-commit Hook (Husky)
#
# 在 commit 前自動執行格式化、代碼檢查、類型檢查、單元測試。
#
# 執行順序：
#   1. format    - 自動格式化代碼（prettier --write）並重新暫存
#   2. lint      - 檢查代碼風格規則（ESLint）
#   3. typecheck - 驗證 TypeScript 型別安全
#   4. test      - 執行單元測試（Vitest）
#   5. openspec  - 驗證 OpenSpec 規格文件
#
# 跳過 hook：git commit --no-verify

# 顏色定義
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m' # No Color

echo "${BLUE}Running pre-commit checks...${NC}"

# =============================================================================
# 前置檢查
# =============================================================================
STAGED_FILES=$(git diff --cached --name-only)
if [ -z "$STAGED_FILES" ]; then
    echo "${YELLOW}No staged files to commit. Skipping pre-commit checks.${NC}"
    exit 0
fi

# =============================================================================
# 代碼格式化
# =============================================================================
echo "${BLUE}[format] Auto-formatting code...${NC}"

if ! bun format; then
    echo "${RED}[format] Format failed!${NC}"
    echo "${YELLOW}Fix: Check Prettier errors and try again${NC}"
    exit 1
fi
echo "${GREEN}[format] Code formatted!${NC}"

# 將格式化後的檔案重新加入暫存區（處理檔名有空格的情況）
echo "${BLUE}[format] Re-staging formatted files...${NC}"
git diff --cached --name-only --diff-filter=ACMR -z | xargs -0 -I {} git add "{}"
echo "${GREEN}[format] Files re-staged!${NC}"

# =============================================================================
# 代碼風格檢查
# =============================================================================
echo "${BLUE}[lint] Running ESLint...${NC}"

if ! bun lint; then
    echo "${RED}[lint] Lint failed!${NC}"
    echo "${YELLOW}Fix: bun lint (check errors above)${NC}"
    exit 1
fi
echo "${GREEN}[lint] Lint passed!${NC}"

# =============================================================================
# TypeScript 類型檢查
# =============================================================================
echo "${BLUE}[typecheck] Running type checks...${NC}"

if ! bun typecheck; then
    echo "${RED}[typecheck] Type check failed!${NC}"
    echo "${YELLOW}Fix: Check TypeScript errors above${NC}"
    exit 1
fi
echo "${GREEN}[typecheck] Type check passed!${NC}"

# =============================================================================
# 單元測試
# =============================================================================
echo "${BLUE}[test] Running unit tests...${NC}"

if ! bunx vitest run; then
    echo "${RED}[test] Tests failed!${NC}"
    echo "${YELLOW}Fix: Check test failures above${NC}"
    exit 1
fi
echo "${GREEN}[test] All tests passed!${NC}"

# =============================================================================
# OpenSpec 規格驗證
# =============================================================================
echo "${BLUE}[openspec] Validating specifications...${NC}"

if ! bunx @fission-ai/openspec validate --specs; then
    echo "${RED}[openspec] Spec validation failed!${NC}"
    echo "${YELLOW}Fix: Check spec format errors above${NC}"
    exit 1
fi
echo "${GREEN}[openspec] All specs validated!${NC}"

# =============================================================================
# 結果
# =============================================================================
echo ""
echo "${GREEN}All pre-commit checks passed!${NC}"
```

## 自訂指引

- **精簡版**：若專案不需要 OpenSpec，移除步驟 5
- **增減檢查項**：依專案需求增減步驟，但建議至少保留 format + lint + typecheck
- **package.json scripts**：確保 `format`、`lint`、`typecheck` 在 `package.json` 的 `scripts` 中有定義
- **Turborepo 專案**：若使用 Turborepo，指令改為 `bun run format`、`bun run lint` 等（透過 turbo 執行）
- **跳過 hook**：緊急情況可用 `git commit --no-verify`，但不建議常態使用

## 來源

`lawyer-app/.husky/pre-commit`
