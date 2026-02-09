---
name: jurislm
description: >-
  This skill should be used when the user asks about JurisLM platform development, architecture, or operations.
  Trigger phrases include: "jurislm_app", "jurislm_cli", "jurislm_db",
  "db status", "db migrate", "db reset", "sync judicial", "sync law", "7-stage pipeline",
  "Unified Agent", "SKILL.md", "tool-registry", "agentic loop", "runUnifiedAgent",
  "shared database", "jurislm_shared_db",
  "Turborepo", "bun workspace", "packages/@jurislm", "Langfuse",
  "TEI embedding", "Ollama", "worktree docker", "pcode", "categories 051-054",
  "constitution rules", "Drizzle ORM", "SSE streaming", "legal RAG",
  "metadata-context", "chunk strategy", "hybrid search", "RRF fusion",
  "bge-m3 embedding", "NAS upload", "Synology", "pipeline performance",
  "launchd", "scheduled sync", "Discord notification", "automation",
  "force reprocess", "resume sync", "checkpoint recovery",
  "taxonomy", "taxonomy build", "synonym expansion", "legal synonyms", "query expansion",
  "concept hierarchy", "broader narrower", "Batch API", "Claude Haiku 4.5", "Claude Sonnet 4",
  "evaluate", "embedding similarity", "baseline evaluation", "baseline-expanded", "Recall@5",
  "knowledge base", "user documents", "Google Drive", "connectors",
  "contract analysis", "document generation", "case management",
  "@jurislm/llm-config", "LLM factory", "createLLM", "MODEL_IDS", "MODEL_PROVIDERS",
  "Extended Thinking", "thinkMode", "thinking budget", "thinkingContent",
  "project documents", "projectContext", "documentSources", "project files",
  "search_knowledge", "parse_contract", "analyze_clause", "render_contract_report",
  "load_playbook", "parse_nda", "check_nda", "render_nda_report",
  "extract_document_info", "generate_legal_document", "sanitize_content",
  "Playbook", "NDA Triage", "Classification", "GREEN", "YELLOW", "RED",
  "escalation triggers", "risk assessment", "lib/agents/shared", "lib/agents/unified".
  Provides comprehensive guidance for Taiwan legal AI platform development.
version: 4.0.0
---

# JurisLM Platform Guide

Taiwan legal AI platform using Bun + Turborepo Monorepo architecture with 3 sub-projects and 9 shared packages.

## Platform Overview

| Component | Technology | Port | Purpose |
|-----------|------------|------|---------|
| jurislm_app | Next.js 16 + shadcn/ui + Anthropic SDK | 3000 | Web application (Legal RAG, Knowledge Base, Contract/Document) |
| jurislm_cli | TypeScript + Bun + Commander.js | - | CLI commands (db, sync, taxonomy, evaluate) |
| jurislm_db | PostgreSQL 18 + pgvector | 5433* | Database Schema |

*Port varies by worktree (main: 5432, plan-a: 5433, plan-b: 5434)

### Shared Services (docker-compose.shared.yml)

```bash
docker compose -f docker-compose.shared.yml up -d
```

| Service | Port | Purpose |
|---------|------|---------|
| jurislm_shared_db | 5440 | PostgreSQL (Judicial + Law + Taxonomy, 19 tables) |
| jurislm-tei-shared | 8090 | TEI Embedding (backup, BAAI/bge-m3) |

### Shared Packages (packages/)

9 internal packages with `@jurislm/` namespace:

| Package | Purpose |
|---------|---------|
| @jurislm/config | Unified environment configuration (Zod validation + Worktree detection) |
| @jurislm/types | Common TypeScript type definitions |
| @jurislm/logging | Structured logging system |
| @jurislm/api-types | API request/response types |
| @jurislm/auth | Authentication module (Judicial API Token) |
| @jurislm/embedding | Embedding factory (TEI/Ollama) |
| @jurislm/errors | Custom error types |
| @jurislm/llm-config | LLM model IDs, providers, validation utilities |
| @jurislm/eslint-config | Shared ESLint configuration |

Reference packages with `workspace:*` in package.json.

## Build System (Turborepo)

Execute unified tasks from root directory:

```bash
bun run lint          # ESLint (all projects, cached)
bun run test          # Tests (all projects, no cache)
bun run typecheck     # TypeScript check (cached)
bun run dev           # Development servers

# Filter specific project
bunx turbo lint --filter=jurislm-cli
bunx turbo test --filter=jurislm-app
```

## Sub-Projects

### jurislm_app (Web Application)

Next.js 16 + React 19 + Drizzle ORM + Anthropic SDK (Unified Agent) + Langfuse:

**App Router Structure** (using `(app)` route group):

| Route | Page | Purpose |
|-------|------|---------|
| `(app)/` | page.tsx | Dashboard / Welcome |
| `(app)/chats` | page.tsx | Conversation list |
| `(app)/knowledge` | page.tsx | Knowledge base overview |
| `(app)/knowledge/[id]` | page.tsx | Document detail |
| `(app)/projects` | page.tsx | Project list |
| `(app)/projects/new` | page.tsx | Create project |
| `(app)/projects/[id]` | page.tsx | Project detail |
| `(app)/projects/[id]/edit` | page.tsx | Edit project |
| `(app)/settings` | page.tsx | Settings home |
| `(app)/settings/connectors` | page.tsx | Google Drive connectors |
| `c/[uuid]` | page.tsx | Conversation (UUID-based URL) |
| `auth/signin` | page.tsx | Sign in |
| `auth/signout` | page.tsx | Sign out |
| `auth/error` | page.tsx | Auth error |

**Core Directories**:

| Directory | Purpose |
|-----------|---------|
| `lib/agents/unified/` | Unified Agent (SKILL.md + tool-registry + agentic loop) |
| `lib/agents/unified/tools/` | 5 tool implementation files (11 tools) |
| `lib/agents/shared/` | Shared modules (classification, llm-factory, prompts) |
| `lib/connectors/` | External connectors (Google Drive OAuth) |
| `lib/knowledge/` | Knowledge base (state machine, processor, storage) |
| `lib/risk/` | Risk assessment (5x5 matrix) |
| `lib/playbook/` | Playbook loader |
| `lib/langfuse/` | LLM observability (Singleton client) |
| `lib/db/` | Drizzle ORM schema and queries |
| `components/chat/` | Chat interface components |

See **`references/jurislm-app.md`** for detailed architecture.

### jurislm_cli (CLI Tool)

7-stage pipeline orchestration with lazy loading architecture:

```
Download -> Unzip -> Parse -> Chunk -> Embed -> Zip -> NAS
```

| Directory | Purpose |
|-----------|---------|
| `src/commands/` | CLI command definitions (lazy loading) |
| `src/commands/impl/` | Command implementations (heavy dependencies) |
| `src/core/` | Pipeline engine, chunk strategies, embedding providers |
| `src/adapters/worker/` | 7-stage executors (Judicial) |
| `src/adapters/law/` | Law sync workers |

**Performance**: Lazy loading reduces `--help` startup time from ~1800ms to ~30ms (**98% improvement**).

**Chunk Strategy**: Metadata Context (only strategy, no LLM calls):
- Extracts JID, JYEAR, JCASE, JTITLE from judgment JSON
- ~0.001 sec/chunk (10,000x faster than LLM-based)
- 100% Recall@5 with Hybrid Search

See **`references/cli-commands.md`** for complete reference.

### jurislm_db (Database)

Two databases with separate migrations:

| Database | Port | Tables | Migrations |
|----------|------|--------|------------|
| jurislm_db | 5433 | Auth + NextAuth + Chat + Knowledge + Contract + Case | `migrations/` (77 files) |
| jurislm_shared_db | 5440 | Judicial + Law + Taxonomy (19 tables) | `migrations-shared/` (15 files) |

**Local Database Tables**:
- Auth: users, accounts, sessions, verification_tokens
- Chat: conversations, messages
- Knowledge: user_folders, user_documents, user_document_chunks, user_embeddings
- Contract: contracts, contract_clauses
- Document: generated_documents
- Case: cases
- Connectors: user_connectors (Google Drive OAuth)

See **`references/database-schema.md`** for table details.

## CLI Commands

Execute from `jurislm_cli` directory:

```bash
cd jurislm_cli

# Database management
bun run src/index.ts db status                    # View migration status
bun run src/index.ts db migrate                   # Execute pending migrations
bun run src/index.ts db status --target shared    # Shared database

# Data synchronization (7-stage pipeline)
bun run src/index.ts sync judicial --category 051  # Judicial sync
bun run src/index.ts sync law --import            # Law sync with DB import
bun run src/index.ts sync law --force             # Force reprocess completed laws
```

See **`references/cli-commands.md`** for complete reference.

## LLM Configuration

### @jurislm/llm-config Package

Cross-project shared LLM configuration:

```typescript
export const MODEL_IDS = {
  HAIKU: "claude-haiku-4-5-20251001",
  SONNET: "claude-sonnet-4-20250514",
  MINISTRAL: "ministral-3:latest",
} as const;

export const MODEL_PROVIDERS: Record<ModelName, LlmProvider> = {
  haiku: "anthropic",
  sonnet: "anthropic",
  ministral: "ollama",
} as const;
```

### Chat Model

| Model | API ID | Use Case |
|-------|--------|----------|
| Claude Haiku 4.5 | claude-haiku-4-5-20251001 | **Default** - Fast response |
| Claude Sonnet 4 | claude-sonnet-4-20250514 | Stronger reasoning |
| Ministral 3 | ministral-3:latest | Local Ollama model |

### Extended Thinking

Enable deeper reasoning for complex legal queries:

| Field | Type | Purpose |
|-------|------|---------|
| `thinkMode` | boolean | Enable Extended Thinking |
| `thinkingBudgetTokens` | number | Token budget (min 1024, default 10000) |
| `thinkingContent` | string | AI reasoning process output |

Extended Thinking allows the model to work through complex legal analysis step-by-step before generating the final response.

### Embedding Provider

| Provider | Default URL | Use Case |
|----------|-------------|----------|
| `ollama` | localhost:11434 | **Primary** - Fast startup |
| `tei` | localhost:8090 | Backup - GPU recommended |

**Model**: BAAI/bge-m3 (1024 dimensions)

## Legal Taxonomy

Query expansion using synonym and hierarchy tables:

### CLI Commands

```bash
cd jurislm_cli

# Build synonyms (generate -> review -> import)
bun run src/index.ts taxonomy build                    # Claude Haiku 4.5 + Batch API
bun run src/index.ts taxonomy build --provider ollama  # Local Ollama

# Query and stats
bun run src/index.ts taxonomy lookup 押金    # Lookup synonyms
bun run src/index.ts taxonomy stats --info   # Show statistics
```

### Performance Impact

| Metric | Baseline | With Expansion | Improvement |
|--------|----------|----------------|-------------|
| Recall@5 | 55% | 75% | **+20%** |

## API Routes

### Core APIs

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/health` | GET | Health check |
| `/api/chat` | POST | SSE chat streaming (Unified Agent) |
| `/api/conversations` | GET/POST | Conversation list/create |
| `/api/conversations/[uuid]` | GET/PATCH/DELETE | Conversation CRUD |
| `/api/conversations/[uuid]/messages` | GET/POST | Messages |
| `/api/conversations/[uuid]/case` | GET/PATCH | Case binding |

### Knowledge Base APIs

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/knowledge/documents` | GET/POST | Document management |
| `/api/knowledge/documents/[id]` | GET | Document detail |
| `/api/knowledge/documents/[id]/retry` | POST | Retry processing |
| `/api/knowledge/documents/process-pending` | POST | Process pending docs |
| `/api/knowledge/folders` | GET/POST | Folder management |
| `/api/knowledge/folders/[id]` | GET/PATCH/DELETE | Folder CRUD |
| `/api/knowledge/search` | GET | Vector search |

### Connector APIs (Google Drive)

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/connectors` | GET | Connector status |
| `/api/connectors/google-drive/authorize` | POST | Start OAuth |
| `/api/connectors/google-drive/callback` | GET | OAuth callback |
| `/api/connectors/google-drive/disconnect` | POST | Disconnect |

### Project APIs

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/projects` | GET/POST | Project list/create |
| `/api/projects/[id]` | GET/PATCH/DELETE | Project CRUD |
| `/api/projects/[id]/documents` | GET/POST | Project documents |

**Project Documents (RAG Integration)**: Project-scoped document retrieval for user-uploaded files:

| Field | Type | Purpose |
|-------|------|---------|
| `projectContext` | ProjectContext | Project ID + User ID scope |
| `documentSources` | SourceCitation[] | Retrieved project documents |

**SourceType**: `"law" | "judgment" | "document"` - Three source types for citations.

### Source Preview APIs

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/sources/[type]/[id]` | GET | Law/Judgment preview data |

### Playbook APIs (Legal Plugin Integration)

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/playbook` | GET | Get project Playbook |
| `/api/playbook` | POST | Upload Playbook |
| `/api/playbook/[id]` | DELETE | Deactivate Playbook |

### NDA Triage APIs

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/nda/triage` | POST | NDA quick screening |

### Risk Assessment APIs

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/risk/assess` | POST | Calculate risk assessment |
| `/api/risk/assessments` | GET | Risk assessment history |

## Legal Plugin Integration

### Unified Agent Architecture

1 Agent + SKILL.md + 11 Tools 取代原 4 LangGraph Agents + keyword router：

**Agentic Loop**: LLM → tool_use → executeTool → tool_result → LLM → ... → text response

**Agent Routing** — LLM auto-detects from first tool call:

| Input Pattern | First Tool Call | Agent Type |
|--------------|----------------|------------|
| Attachment + NDA/保密 | `parse_nda` | nda |
| Attachment + 合約/契約 | `parse_contract` | contract |
| 草擬/撰寫 keywords | `extract_document_info` | document |
| Other legal questions | `search_knowledge` | rag |

**11 Tools**:

| Tool | Purpose |
|------|---------|
| `search_knowledge` | 三層知識庫 RAG 檢索（共用 + 個人 + 專案） |
| `parse_contract` | 解析合約文本，提取 metadata 和 clauses |
| `analyze_clause` | 分析條款風險，產生 classification |
| `render_contract_report` | 渲染合約分析 Markdown 報告 |
| `load_playbook` | 載入專案 Playbook 審核標準 |
| `parse_nda` | 解析 NDA 文本 |
| `check_nda` | NDA 10 點檢查 |
| `render_nda_report` | 渲染 NDA 審核報告 |
| `extract_document_info` | 提取文件生成所需資訊 |
| `generate_legal_document` | 生成法律文件 |
| `sanitize_content` | 淨化使用者輸入（防 prompt injection） |

### Classification System

三級分類系統（GREEN/YELLOW/RED）：

| Classification | Description | Action |
|----------------|-------------|--------|
| GREEN | 低風險 | 可直接進行 |
| YELLOW | 中等風險 | 需要審查 |
| RED | 高風險 | 必須升級 |

**與 RiskLevel (4 級) 的關係**：
- RiskLevel (GREEN/YELLOW/ORANGE/RED) 用於 5×5 風險矩陣評估
- Classification (GREEN/YELLOW/RED) 用於條款審核決策
- Contract Agent 直接輸出 Classification，不需要中間轉換

### Playbook Configuration

Playbook 使用 Markdown + YAML frontmatter 格式：

```markdown
---
name: NDA Playbook
version: 1.0
scope: nda
---

## 機密資訊定義

### 審核重點
- 確認範圍明確
- 檢查排除條款
```

**載入方式**：Playbook 不參與三層知識庫 RAG 檢索，獨立載入。

### Escalation Triggers

升級觸發器類型：

| Type | Description | Example |
|------|-------------|---------|
| keyword | 關鍵字匹配 | 刑事、詐欺、違法 |
| threshold | 閾值比較 | riskScore > 80 |
| missing | 缺欄位檢查 | 無保密期限條款 |
| composite | 複合條件 | AND/OR 組合 |

### Shared Agent Modules

`lib/agents/shared/` 提供共用邏輯：

| Module | Purpose |
|--------|---------|
| classification.ts | Classification 類型和轉換函數 |
| llm-factory.ts | LLM factory（Anthropic SDK） |
| prompts.ts | 共用 prompt templates |

### NDA Triage

快速篩選 NDA 的 Unified Agent 流程：

**流程**：parse_nda → check_nda → render_nda_report

**10 點檢查**：
1. 機密資訊定義
2. 保密期間
3. 排除條款
4. 義務範圍
5. 違約責任
6. 管轄法院
7. 終止條款
8. 雙方權利義務
9. 爭議解決
10. 其他特殊條款

## Key Concepts

**pcode**: MOJ stable identifier from law URLs. Use for unique law identification.

**Category Codes (051-054)**:
| Code | Name | Table |
|------|------|-------|
| 051 | 裁判書 | documents_051 |
| 052 | 大法官解釋 | documents_052 |
| 053 | 憲法法庭判決 | documents_053 |
| 054 | 憲法法庭裁定 | documents_054 |

**Constitution Rules**: 13 core rules (v4.0.0) in `.specify/memory/constitution.md`.

## Development Workflow

### Pre-Development Checks

1. Search existing code to avoid duplication
2. Verify database schema with `\d table_name`
3. Check Git history for partial implementations

### Git Workflow (Monorepo)

Execute all Git operations from root directory:

```bash
git add jurislm_cli/src jurislm_app/lib
git commit -m "feat: cross-project feature"
```

### Worktree Docker

Create independent Docker environments per worktree. See **`references/worktree-docker.md`** for port mapping.

## Scheduled Sync Automation

Automated synchronization using macOS launchd with Discord notifications.

| Schedule | Category | Time | plist |
|----------|----------|------|-------|
| Weekly | 051 裁判書 | Sun 02:00 | `com.jurislm.sync-051.plist` |
| Daily | 052-054 | Daily 03:00 | `com.jurislm.sync-daily.plist` |

See **`references/scheduled-sync.md`** for complete setup.

## Reference Files

- **`references/jurislm-app.md`** - Web application architecture (Unified Agent)
- **`references/cli-commands.md`** - Complete CLI reference
- **`references/database-schema.md`** - Table relationships
- **`references/judicial-sync.md`** - 7-stage pipeline
- **`references/judicial-json-structure.md`** - Judicial JSON schema
- **`references/worktree-docker.md`** - Port mapping
- **`references/law-json-structure.md`** - MOJ API JSON schema
- **`references/scheduled-sync.md`** - Scheduled automation
