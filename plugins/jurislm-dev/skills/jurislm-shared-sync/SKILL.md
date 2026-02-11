---
name: jurislm-shared-sync
description: >-
  This skill should be used when the user asks to "sync shared database",
  "synchronize jurislm_shared_db", "full database sync", "complete shared db sync",
  "sync judicial and law data", "refresh shared database", "initialize shared db",
  "run full sync", "populate shared database", "setup jurislm_shared_db from scratch",
  "reset and resync shared db", "full judicial sync", "complete law import",
  "同步共用資料庫", "初始化共用資料庫", "完整同步", "同步司法資料",
  "sync all categories", "sync 051-054", "import law data to shared".
  Provides step-by-step workflow to complete full synchronization of jurislm_shared_db
  including Judicial (051-054), Law, and Taxonomy data.
version: 1.3.0
---

# Shared Database Full Sync Workflow

Synchronize jurislm_shared_db (PostgreSQL 18 + pgvector) with judicial, law, and taxonomy data.

## Quick Start

For experienced users, execute these commands in order from `jurislm_cli`:

```bash
cd jurislm_cli
bun run src/index.ts db migrate --target shared        # Step 1: Apply migrations
bun run src/index.ts sync judicial                     # Step 2: Judicial 9-stage pipeline (auto DB upload + cleanup)
bun run src/index.ts sync law                          # Step 3: Law 9-stage pipeline (auto import + cleanup)
bun run src/index.ts taxonomy build                    # Step 4: Synonyms (optional)
bun run src/index.ts db status --target shared --info  # Step 5: Verify
```

For detailed guidance, continue reading below.

## Complete Table Structure (25 Tables)

### System (1 table)
| Table | Purpose | Data Source |
|-------|---------|-------------|
| `migrations` | Migration tracking | `db migrate --target shared` |

### Judicial (13 tables)

*Note: Document counts are estimates and vary with Judicial Yuan data releases.*

| Table | Purpose | Data Source |
|-------|---------|-------------|
| `categories` | 4 fixed categories | Migration auto-INSERT |
| `datasets` | Dataset metadata | `sync judicial` |
| `filesets` | Fileset tracking | `sync judicial` |
| `fileset_items` | Fileset 內個別檔案追蹤 | `sync judicial` |
| `documents_051` | 裁判書 (~21M) | `sync judicial --category 051` |
| `documents_052` | 大法官解釋 (~800) | `sync judicial --category 052` |
| `documents_053` | 憲法法庭判決 (~300) | `sync judicial --category 053` |
| `documents_054` | 憲法法庭裁定 (~500) | `sync judicial --category 054` |
| `document_embeddings_051` | 裁判書向量 | `sync judicial --category 051` |
| `document_embeddings_052` | 解釋向量 | `sync judicial --category 052` |
| `document_embeddings_053` | 判決向量 | `sync judicial --category 053` |
| `document_embeddings_054` | 裁定向量 | `sync judicial --category 054` |
| `document_embedding_status` | Embedding 處理狀態追蹤 | `sync judicial` |

### Law (4 tables)
| Table | Purpose | Data Source |
|-------|---------|-------------|
| `laws` | 法規主表 (~11,737) | `sync law` (IMPORT phase) |
| `law_articles` | 法規條文 | `sync law` (IMPORT phase) |
| `law_attachments` | 法規附件 | `sync law` (IMPORT phase) |
| `law_embeddings` | 法規向量 | `sync law` (IMPORT phase) |

### Taxonomy (3 tables)
| Table | Purpose | Data Source |
|-------|---------|-------------|
| `legal_synonyms` | 法律同義詞 | `taxonomy build` |
| `legal_synonyms_meta` | 同義詞元資料 | `taxonomy build` |
| `legal_concept_hierarchy` | 概念階層 (DAG) | `taxonomy build` |

### Evaluation (2 tables)
| Table | Purpose | Data Source |
|-------|---------|-------------|
| `chunk_evaluation_runs` | 評估執行記錄 | `evaluate` |
| `chunk_evaluation_questions` | 評估測試問題 | `evaluate` |

### System Tracking (2 tables)
| Table | Purpose | Data Source |
|-------|---------|-------------|
| `processing_jobs` | 處理任務追蹤 | `sync judicial` / `sync law` |
| `deleted_judgments` | 已刪除裁判追蹤 | `sync judicial` |

## Table Dependencies (Sync Order)

```
Step 1: migrations (system)
    └── db migrate --target shared

Step 2: categories (4 fixed records, auto-inserted by migration)

Step 3: Judicial (dependency chain)
    categories → datasets → filesets → documents_* → document_embeddings_*
    └── sync judicial (handles dependencies automatically)

Step 4: Law (dependency chain)
    laws → law_articles, law_attachments, law_embeddings
    └── sync law (auto IMPORT + CLEANUP)

Step 5: Taxonomy (independent, no dependencies)
    legal_synonyms, legal_concept_hierarchy
    └── taxonomy build
```

## Prerequisites Checklist

Verify before starting:

1. **Docker services running**:
   ```bash
   docker compose -f docker-compose.shared.yml up -d
   docker compose -f docker-compose.shared.yml ps
   ```

2. **Ollama available** (for embedding):
   ```bash
   curl -s http://localhost:11434/api/tags | head -1
   ```

3. **Database connection**:
   ```bash
   cd jurislm_cli
   bun run src/index.ts db status --target shared
   ```

4. **Environment variables** (`.env.shared`):
   - `SHARED_DATABASE_URL`
   - `OLLAMA_BASE_URL`
   - `SYNOLOGY_BASE_URL`, `SYNOLOGY_ACCOUNT`, `SYNOLOGY_PASSWORD`

## Sync Workflow (4 Steps)

Execute from `jurislm_cli` directory:

```bash
cd jurislm_cli
```

### Step 1: Database Migration

Apply pending migrations (creates all 25 tables + inserts 4 categories):

```bash
bun run src/index.ts db status --target shared    # Check status
bun run src/index.ts db migrate --target shared   # Apply migrations
```

### Step 2: Judicial Sync (051-054)

Sync all 4 judicial categories (9-stage pipeline, auto DB upload + cleanup):

```bash
# Full pipeline (from API download to DB upload + cleanup)
bun run src/index.ts sync judicial

# Or sync specific categories
bun run src/index.ts sync judicial --category 051    # 裁判書
bun run src/index.ts sync judicial --category 052    # 大法官解釋
bun run src/index.ts sync judicial --category 053    # 憲法法庭判決
bun run src/index.ts sync judicial --category 054    # 憲法法庭裁定

# DB upload only (if pipeline completed but DB upload was skipped)
bun run src/index.ts sync judicial --mode db_upload
```

**Full Pipeline**: Download → Unzip → Parse → Chunk → Embed → Zip → NAS → DB_Upload → Cleanup

### Step 3: Law Sync

Sync all laws (9-stage pipeline, auto import + cleanup):

```bash
# Full pipeline (from API download to DB import + cleanup)
bun run src/index.ts sync law

# Import only (if pipeline completed but import was skipped)
bun run src/index.ts sync law --mode import

# Cleanup only (remove intermediate files)
bun run src/index.ts sync law --mode cleanup
```

**Pipeline**: Download → Unzip → Parse → Chunk → Embed → Zip → NAS → Import → Cleanup

### Step 4: Taxonomy Build (Optional)

Generate legal synonyms for query expansion. Skip if synonyms already exist.

```bash
bun run src/index.ts taxonomy build    # Uses Claude Haiku 4.5 Batch API
```

Requires `ANTHROPIC_API_KEY` in `.env.shared`.

### Step 4b: Verification

Verify all 25 tables have data:

```bash
bun run src/index.ts db status --target shared --info
```

Expected output shows table counts for all 25 tables. Critical tables to verify:
- `categories`: 4 (fixed)
- `datasets`, `filesets`: > 0
- `documents_051`: > 0 (largest table)
- `laws`: ~11,737

For complete verification SQL queries, see **`references/sync-workflow.md`**.

## Common Options

| Option | Purpose |
|--------|---------|
| `--info` | Verbose output for debugging |
| `--limit N` | Process only N items (for testing) |
| `--mode embed` | Run only embed phase |
| `--mode db_upload` | Run only database upload (Judicial) |
| `--mode import` | Run only database import (Law, or alias for db_upload in Judicial) |
| `--mode cleanup` | Run only cleanup phase |
| `--force` | Force reprocess completed items |
| `--resume` | Resume from checkpoint after interruption |
| `--dataset ID` | Filter specific dataset (Judicial) |
| `--fileset ID` | Filter specific fileset (Judicial) |

## Expected Duration

| Component | Estimated Time | Tables Populated |
|-----------|---------------|------------------|
| Migrations | < 1 min | migrations, categories |
| Judicial 051 | 2-4 hours | datasets, filesets, documents_051, document_embeddings_051 |
| Judicial 052-054 | 30 min each | documents_052-054, document_embeddings_052-054 |
| Law sync | 1-2 hours | laws, law_articles, law_attachments, law_embeddings |
| Taxonomy | 10-30 min | legal_synonyms, legal_concept_hierarchy |

**Total**: 4-8 hours for complete fresh sync (all 25 tables)

## Reference Files

For detailed information:
- **`references/sync-workflow.md`** - Complete command reference, all options, verification SQL
- **`references/troubleshooting.md`** - Common issues and solutions
