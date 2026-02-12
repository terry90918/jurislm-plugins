# JurisLM Plugins Official

Claude Code 外掛市集，提供 MCP 伺服器、Skills 與跨專案開發經驗模式。

## 安裝

新增市集：

```
/plugin marketplace add terry90918/jurislm-plugins
```

安裝外掛：

```
/plugin install <plugin-name>@jurislm-plugins
```

## 可用外掛

### MCP Server 類型

#### hetzner

透過 Claude Code 管理 Hetzner Cloud 基礎設施的 MCP 伺服器（14 個工具）。

**功能：**
- 列出、建立、刪除伺服器
- 開機、關機、重新啟動伺服器
- 建立和管理 SSH 金鑰
- 查看可用映像檔、伺服器類型和資料中心位置

**設定步驟：**

1. 從 [Hetzner Cloud Console](https://console.hetzner.cloud/) → 專案 → Security → API Tokens 取得 API Token

2. 新增至 `~/.zshenv`：
   ```bash
   export HETZNER_API_TOKEN="your_token_here"
   ```

3. 重新啟動 Claude Code

#### coolify

透過 Claude Code 管理 Coolify 自託管 PaaS 的 MCP 伺服器（35 個最佳化工具）。

**功能：**
- 智能查詢（支援域名和 IP，不只是 UUID）
- 完整基礎設施管理（應用、資料庫、服務）
- 診斷和問題掃描
- 批量操作支援

**設定步驟：**

1. 從 Coolify Dashboard → Settings → API Tokens 取得 API Token

2. 新增至 `~/.zshenv`：
   ```bash
   export COOLIFY_ACCESS_TOKEN="your-api-token"
   export COOLIFY_BASE_URL="https://your-coolify-instance.com"
   ```

3. 重新啟動 Claude Code

### Skill Only 類型

#### jurislm-dev

JurisLM 台灣法律 AI 平台開發 Skill（3 個 skills）。

**涵蓋：**
- Unified Agent（SKILL.md + 11 Tools agentic loop）
- Admin Dashboard（Hono + React 19 + Vite 7）
- CLI 工具（db、sync、taxonomy、evaluate）
- jurislm_shared_db 完整同步工作流
- 法律同義詞生成與匯入

#### lawyer

劉尹惠律師事務所網站開發 Skill。

**涵蓋：**
- Payload CMS（Collection 設計、Migration、Admin CSS）
- Coolify 部署工作流（Staging/Production）
- Playwright E2E 測試架構與最佳實踐
- Ghost CMS → Payload CMS 內容遷移

#### stock

台股看盤應用開發 Skill。

**涵蓋：**
- TWSE/Yahoo Finance API 整合模式
- 47+ API endpoints 參考文件
- 40+ Custom hooks 使用指南
- Prisma 資料庫 schema
- E2E 測試與單元測試策略

#### github-release

GitHub Actions 標準化工作流 Skill。

**涵蓋：**
- Husky pre-commit hook（格式化、lint、typecheck、test）
- Release Please 自動版本管理
- Claude Code GitHub Action 配置
- Claude Code 自動 PR Review
- GitHub Release Notes 分類配置

#### lessons-learned

跨專案開發經驗模式庫（66 個模式，12 個分類）。

**涵蓋：**
- 診斷除錯（8）、測試策略（15）、基礎設施與部署（7）
- 安全與錯誤處理（6）、業務邏輯（4）、架構與重構（5）
- Git 工作流（3）、工具與工作流（2）、雲端遷移與環境配置（6）
- 前端工具鏈與框架（3）、Turborepo Docker 部署（4）、資料匯入與 Migration（3）

## 授權

MIT
