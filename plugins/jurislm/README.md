# JurisLM Plugin

JurisLM 台灣法律 AI 平台（jurislm.com）的 Claude Code 開發 Skill。

## Skills

| Skill | 觸發詞 | 用途 |
|-------|--------|------|
| jurislm | jurislm_app, Unified Agent, tool-registry, db migrate, sync judicial... | 平台開發全面指南 |
| jurislm-shared-sync | sync shared database, 同步共用資料庫, sync 051-054... | jurislm_shared_db 完整同步工作流 |
| jurislm-taxonomy-generate | build synonyms, 生成同義詞, taxonomy build... | 法律同義詞生成與匯入 |

## 安裝

```
/plugin install jurislm@jurislm-plugins
```

## 適用場景

- 開發 Unified Agent（SKILL.md + 11 Tools agentic loop）
- 操作 CLI 工具（db、sync、taxonomy、evaluate）
- 同步 jurislm_shared_db（Judicial 051-054 + Law + Taxonomy）
- 生成法律同義詞資料庫
- 理解三面板 UI 架構與 SSE 串流

## 技術棧

| 類別 | 技術 |
|------|------|
| 框架 | Next.js 16 (App Router) + React 19 |
| AI | Anthropic SDK (Unified Agent) + Langfuse |
| 資料庫 | PostgreSQL 18 + pgvector + Drizzle ORM |
| CLI | Bun + Commander.js |
| 測試 | Vitest + Playwright |
| 套件管理 | Bun + Turborepo |

## 相關連結

- [JurisLM Dashboard](https://dashboard.jurislm.com)
- [GitHub Repo](https://github.com/terry90918/jurislm)

## 授權

MIT
