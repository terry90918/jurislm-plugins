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
bun run src/index.ts sync judicial                     # Step 2a: Judicial pipeline
bun run src/index.ts sync judicial --mode import       # Step 2b: Import to DB
bun run src/index.ts sync law                          # Step 3a: Law pipeline
bun run src/index.ts sync law --mode import            # Step 3b: Import to DB
bun run src/index.ts taxonomy build                    # Step 4: Synonyms (optional)
bun run src/index.ts db status --target shared --info  # Step 5: Verify
```

For detailed guidance, continue reading below.

## Complete Table Structure (19 Tables)

### System (1 table)
| Table | Purpose | Data Source |
|-------|---------|-------------|
| `migrations` | Migration tracking | `db migrate --target shared` |

### Judicial (11 tables)

*Note: Document counts are estimates and vary with Judicial Yuan data releases.*

| Table | Purpose | Data Source |
|-------|---------|-------------|
| `categories` | 4 fixed categories | Migration auto-INSERT |
| `datasets` | Dataset metadata | `sync judicial` |
| `filesets` | Fileset tracking | `sync judicial` |
| `documents_051` | 裁判書 (~21M) | `sync judicial --category 051` |
| `documents_052` | 大法官解釋 (~800) | `sync judicial --category 052` |
| `documents_053` | 憲法法庭判決 (~300) | `sync judicial --category 053` |
| `documents_054` | 憲法法庭裁定 (~500) | `sync judicial --category 054` |
| `document_embeddings_051` | 裁判書向量 | `sync judicial --category 051` |
| `document_embeddings_052` | 解釋向量 | `sync judicial --category 052` |
| `document_embeddings_053` | 判決向量 | `sync judicial --category 053` |
| `document_embeddings_054` | 裁定向量 | `sync judicial --category 054` |

### Law (4 tables)
| Table | Purpose | Data Source |
|-------|---------|-------------|
| `laws` | 法規主表 (~11,737) | `sync law --mode import` |
| `law_articles` | 法規條文 | `sync law --mode import` |
| `law_attachments` | 法規附件 | `sync law --mode import` |
| `law_embeddings` | 法規向量 | `sync law --mode import` |

### Taxonomy (2 tables)
| Table | Purpose | Data Source |
|-------|---------|-------------|
| `legal_synonyms` | 法律同義詞 | `taxonomy build` |
| `legal_concept_hierarchy` | 概念階層 (DAG) | `taxonomy build` |

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
    └── sync law + sync law --mode import

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
   - `NAS_HOST`, `NAS_PORT`, `NAS_USERNAME`, `NAS_PASSWORD`

## Sync Workflow (5 Steps)

Execute from `jurislm_cli` directory:

```bash
cd jurislm_cli
```

### Step 1: Database Migration

Apply pending migrations (creates all 19 tables + inserts 4 categories):

```bash
bun run src/index.ts db status --target shared    # Check status
bun run src/index.ts db migrate --target shared   # Apply migrations
```

### Step 2: Judicial Sync (051-054)

Sync all 4 judicial categories (7-stage pipeline + import):

```bash
# Option A: Full pipeline (from API download to NAS upload)
bun run src/index.ts sync judicial

# Option B: Import only (if data already exists locally with embeddings)
bun run src/index.ts sync judicial --mode import

# Or sync specific categories
bun run src/index.ts sync judicial --category 051    # 裁判書
bun run src/index.ts sync judicial --category 052    # 大法官解釋
bun run src/index.ts sync judicial --category 053    # 憲法法庭判決
bun run src/index.ts sync judicial --category 054    # 憲法法庭裁定
```

**Full Pipeline**: Download → Unzip → Parse → Chunk → Embed → Zip → NAS
**Import Mode**: Reads from local `extend/` and `*_embed/` directories → Import to DB

### Step 3: Law Sync

Sync all laws (7-stage pipeline) then import to database:

```bash
# Option A: Full pipeline then import (if no local data)
bun run src/index.ts sync law                   # Download → Parse → Chunk → Embed → Zip → NAS
bun run src/index.ts sync law --mode import     # Import to DB

# Option B: Import only (if data already exists locally with embeddings)
bun run src/index.ts sync law --mode import
```

**Pipeline**: Download → Parse → Chunk → Embed → Zip → NAS

### Step 4: Taxonomy Build (Optional)

Generate legal synonyms for query expansion. Skip if synonyms already exist.

```bash
bun run src/index.ts taxonomy build    # Uses Claude Haiku 4.5 Batch API
```

Requires `ANTHROPIC_API_KEY` in `.env.shared`.

### Step 5: Verification

Verify all 19 tables have data:

```bash
bun run src/index.ts db status --target shared --info
```

Expected output shows table counts for all 19 tables. Critical tables to verify:
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
| `--mode import` | Run only database import (Judicial/Law) |
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

**Total**: 4-8 hours for complete fresh sync (all 19 tables)

## Reference Files

For detailed information:
- **`references/sync-workflow.md`** - Complete command reference, all options, verification SQL
- **`references/troubleshooting.md`** - Common issues and solutions
