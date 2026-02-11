# Judicial Document Synchronization

Complete workflow for synchronizing judicial documents from Taiwan Judicial Yuan Open Data Platform.

## Overview

Execute a nine-phase synchronization pipeline:

1. **Download** - Fetch compressed filesets from Judicial Yuan API
2. **Unzip** - Extract files (auto-detect ZIP/7Z/RAR)
3. **Parse** - PDF parsing (attorneys, justices, opinions for 052-054)
4. **Chunk** - Metadata Context text splitting (no LLM calls)
5. **Embed** - Generate vectors via BAAI/bge-m3 (1024 dim)
6. **Zip** - Compress output files
7. **NAS** - Upload to Synology NAS
8. **DB_Upload** - Upload documents and embeddings to shared database
9. **Cleanup** - Remove intermediate files (extend/, embeddings.jsonl.gz)

### Metadata Context (CHUNK Phase)

The CHUNK phase uses **Metadata Context** strategy:

| Setting | Value |
|---------|-------|
| Strategy | `metadata-context` (only option) |
| Context Source | JSON metadata (JID, JYEAR, JCASE, JTITLE, court_name) |
| Text Splitter | @langchain/textsplitters RecursiveCharacterTextSplitter |
| Chunk Size | 4000 characters |
| Chunk Overlap | 400 characters |
| Performance | ~0.001 sec/chunk (10,000x faster than LLM-based) |
| Accuracy | 100% Recall@5 with Hybrid Search |

**How it works**:
1. Extracts structured metadata from judgment JSON (JID, JYEAR, JCASE, JTITLE)
2. Generates context prefix from metadata (no LLM inference needed)
3. Context is prepended to chunk text before embedding
4. Combines with JID exact matching + vector search + RRF fusion for high accuracy

**Breaking Change**: `--strategy contextual` is no longer supported. Only `metadata-context` is available.

## Quick Start

```bash
cd jurislm_cli

# Sync all categories (default behavior)
bun run src/index.ts sync judicial

# Execute sync for specific category
bun run src/index.ts sync judicial --category 051

# Limit number of judgments (for testing)
bun run src/index.ts sync judicial --category 051 --limit 10

# Sync multiple categories
bun run src/index.ts sync judicial --category 051,052,053,054

# Verbose output
bun run src/index.ts sync judicial --category 051 --info
```

## Pre-flight Checks

### 1. System Health

```bash
docker compose ps
```

**Required Services**:
- PostgreSQL (jurislm_db, port 5433) - healthy
- Embedding service - required for vector generation:
  - Ollama (port 11434) if `EMBEDDING_PROVIDER=ollama` (default)
  - TEI (port 8090) if `EMBEDDING_PROVIDER=tei`
- Shared database (port 5442) - required for data storage

**Note**: Text chunking is performed locally using @langchain/textsplitters (no external service required).

**Embedding Provider Health Check**:
```bash
# For TEI
curl -s http://localhost:8090/health | jq

# For Ollama
curl -s http://localhost:11434/api/tags | jq
```

**Checkpoint**: Stop if any critical service is unhealthy.

### 2. Database Validation

```bash
docker exec jurislm_shared_db psql -U postgres -d jurislm_shared_db -c "\dt documents_*"
```

**Required Tables** (in jurislm_shared_db):
- `documents_{category}` (e.g., documents_051)
- `document_embeddings_{category}`
- `categories`, `datasets`, `filesets`

### 3. Current Data Status

```bash
docker exec jurislm_shared_db psql -U postgres -d jurislm_shared_db -c "
SELECT 'documents_051' as tbl, COUNT(*) FROM documents_051
UNION ALL SELECT 'document_embeddings_051', COUNT(*) FROM document_embeddings_051;
"
```

Record counts for post-sync verification.

## Command Options

| Option | Description | Default |
|--------|-------------|---------|
| `--category <list>` | Comma-separated category IDs (051,052,053,054) | all |
| `--mode {phase}` | Execute only specified phase (download/unzip/parse/chunk/embed/zip/nas/db_upload/import/cleanup) | all |
| `--strategy {name}` | Chunk strategy (metadata-context only) | metadata-context |
| `--limit N` | Limit number of **judgments** to process (cross-fileset counting) | unlimited |
| `--dataset ID` | Process only specified dataset ID | - |
| `--fileset ID` | Process only specified fileset ID (requires --dataset) | - |
| `--force` | Force reprocess completed filesets | false |
| `--info` | Verbose output | false |
| `--categories` | Download and display all categories | false |

## NAS Configuration (Synology)

The NAS phase uploads embeddings to Synology NAS (replaced GCS in 2026-01):

| Setting | Description |
|---------|-------------|
| `SYNOLOGY_BASE_URL` | NAS QuickConnect URL (e.g., `https://xxx.quickconnect.to:5001`) |
| `SYNOLOGY_ACCOUNT` | NAS account username |
| `SYNOLOGY_PASSWORD` | NAS account password |

**NAS Upload Path**:
```
/home/jurislm-embedding/{category}/{dataset}/{fileset}/embeddings.jsonl.gz
```

**Behavior**:
- If `SYNOLOGY_BASE_URL` is not set, NAS upload is skipped
- Automatic retry with exponential backoff (3 attempts)
- Creates parent directories automatically

**Backward Compatibility**: The `--mode gcs` option is still accepted as an alias for `--mode nas`.

## Progress Monitoring

Watch for:
- **jid-level progress** - Individual document tracking during chunk phase
- Timeout warnings (50%, 80%, 90%)
- Failed items count
- Pipeline statistics (completed/partial/failed)
- Fileset completion statistics with timing data

### Check status.json

Each fileset maintains a `status.json` file tracking pipeline progress:

```bash
# View status for a specific fileset
cat /path/to/fileset/status.json | jq
```

**Status file contents**:
- `phases` - Completion status for each of 9 phases
- `total_count` - Total documents in fileset
- `completed_count` - Processed documents
- `error_count` - Failed documents

## Data Validation

### Document Integrity

Verify required fields:
- `jid` - 100% required
- `fileset_id` - 100% required
- `jfull` content - >95%
- `jdate` - >90%
- `court_name` - >90%

### Embedding Coverage

Check embedding-to-document ratio:
- Coverage should be >= 99%
- Each document needs at least 1 embedding chunk
- Embedding dimension: 1024 (BAAI/bge-m3)

### Sync Progress

Verify final status in `status.json` files:
- All phases should show `completed`
- `skipped` acceptable for resume mode
- Error count should be < 1%

## Post-Sync Actions

```bash
# Update database statistics
docker exec jurislm_shared_db psql -U postgres -d jurislm_shared_db -c "
ANALYZE documents_051;
ANALYZE document_embeddings_051;
"

# Verify HNSW index
docker exec jurislm_shared_db psql -U postgres -d jurislm_shared_db -c "
SELECT indexname, pg_size_pretty(pg_relation_size(indexname::regclass))
FROM pg_indexes WHERE tablename = 'document_embeddings_051' AND indexdef LIKE '%hnsw%';
"
```

## Checkpoint Reference

| Phase | Checkpoint | Action on Failure |
|-------|------------|-------------------|
| Pre-flight | System Health | Stop, fix services |
| Pre-flight | Schema Validation | Stop, run migrations |
| Sync | Preview Review | Stop if unexpected |
| Sync | Progress | Automatic resume (no action needed) |
| Validation | Document Integrity | Investigate data |
| Validation | Embedding Coverage | Re-run embed step |

## Error Recovery

### Download Failures

```bash
cd jurislm_cli
bun run src/index.ts sync judicial --category 051 --mode download --fileset {FILESET_ID}
```

### Embedding Failures

```bash
# Check embedding provider configuration
echo $EMBEDDING_PROVIDER  # Should be 'tei' or 'ollama'

# Check TEI service health (if EMBEDDING_PROVIDER=tei)
curl -s http://localhost:8090/health

# Check Ollama service health (if EMBEDDING_PROVIDER=ollama)
curl -s http://localhost:11434/api/tags

# Start Ollama if not running
ollama serve

# Retry embedding phase only
cd jurislm_cli
bun run src/index.ts sync judicial --category 051 --mode embed
```

## Performance Benchmarks

Tested with fileset 58452 (Category 051, 1,130 documents, 1,197 chunks):

| Phase | Duration | Throughput | Notes |
|-------|----------|------------|-------|
| DOWNLOAD | ~1s | 3.25 MB/s | 2.6 MB compressed |
| UNZIP | ~0.03s | - | Skipped if already extracted |
| CHUNK | **~0.5s** | 2,260 docs/s | **No LLM, metadata-based** |
| EMBED | ~6min | 3.4 emb/s | 1,197 embeddings |
| ZIP | ~1s | - | 8.3 MB output |
| NAS | ~3s | 2.8 MB/s | Upload to Synology |

### Performance Improvement (v2.53.0)

**CHUNK phase improved from ~2.5h to ~0.5s (18,000x faster)**:
- Removed LLM dependency (no more ministral-3:3b calls)
- Uses structured JSON metadata for context prefix
- No inference latency, pure CPU computation

### Category 051 Full Sync Estimate (Updated)

| Metric | Previous (LLM) | Current (metadata-context) |
|--------|----------------|---------------------------|
| Total documents | ~46.6M | ~46.6M |
| CHUNK time/doc | ~8 sec | ~0.001 sec |
| CHUNK time (total) | ~12 年 | **< 13 小時** |
| Total sync time | 12+ 年 | **~1-2 天** (EMBED 為瓶頸) |

**New Bottleneck**: EMBED phase (3.4 emb/s) is now the limiting factor.

## Category Reference

| Category | Description | Typical Count |
|----------|-------------|---------------|
| 051 | 裁判書 | **~46.6M** documents (實測) |
| 052 | 大法官解釋 | ~800 documents |
| 053 | 憲法法庭判決 | ~300 documents |
| 054 | 憲法法庭裁定 | ~500 documents |

## Pipeline Architecture

```
+--------------------------------------------------------------------------------------------------------------+
|                                           SyncPipeline                                                        |
|  +----------+ +--------+ +-------+ +-------+ +-------+ +-----+ +-----+ +-----------+ +---------+            |
|  | Download |>| Unzip  |>| Parse |>| Chunk |>| Embed |>| Zip |>| NAS |>| DB_Upload |>| Cleanup |            |
|  +----------+ +--------+ +-------+ +-------+ +-------+ +-----+ +-----+ +-----------+ +---------+            |
|       |           |          |         |         |        |       |          |             |                  |
|       v           v          v         v         v        v       v          v             v                  |
|  +----------------------------------------------------------------------------------------------------------+|
|  |                              status.json (9-stage tracking)                                               ||
|  +----------------------------------------------------------------------------------------------------------+|
+--------------------------------------------------------------------------------------------------------------+
```

**DB_Upload**: Uses `importCategory()` with dataset/fileset filters to ensure FK constraints (categories → datasets → filesets → documents → embeddings). All INSERTs use ON CONFLICT UPSERT for idempotent re-execution.

**Cleanup**: Removes intermediate files (`extend/` directory and `embeddings.jsonl.gz`) while preserving `status.json` and archive files for checkpoint tracking.

## Environment Variables

```bash
# Required - Judicial Yuan API
JUDICIAL_USERNAME=your_username
JUDICIAL_PASSWORD=your_password
SHARED_DATABASE_URL=postgresql://postgres:<password>@46.225.58.202:5442/jurislm_shared_db

# Embedding Provider Configuration (Required)
# ollama = primary (default), tei = backup
EMBEDDING_PROVIDER=ollama       # Options: ollama | tei

# Embedding URL (Optional - has sensible defaults per provider)
# Ollama default: http://localhost:11434
# TEI default: http://localhost:8090
EMBEDDING_URL=http://localhost:11434

# NAS Configuration (Synology) - Optional
# If not set, NAS upload phase is skipped
SYNOLOGY_BASE_URL=https://xxx.quickconnect.to:5001
SYNOLOGY_ACCOUNT=your_account
SYNOLOGY_PASSWORD=your_password
```

### Embedding Provider Selection

| Provider | Default URL | Batch Size | Use Case |
|----------|-------------|------------|----------|
| `ollama` | http://localhost:11434 | 16 | **Primary** - Fast startup, recommended |
| `tei` | http://localhost:8090 | 32 | Backup - GPU recommended |

**Ollama** (Primary - Recommended):
- Fast startup (seconds)
- Default provider for daily development
- BAAI/bge-m3 model (1024 dimensions)
- Smaller batch size (16) for quality preservation

**TEI** (Backup - Text Embeddings Inference):
- Better for production with GPU acceleration
- Longer cold start (~7 min on CPU)
- Higher throughput with larger batch sizes

**Note**: Context generation now uses structured JSON metadata (no LLM calls), independent of embedding provider.

## Disk Space Management

The pipeline automatically cleans up intermediate files:
- Downloaded ZIP files: Deleted after extraction
- Extracted JSON files: Deleted after import
- Embedding chunks: Compressed and uploaded to NAS

**Estimated space requirements**:
- Category 051: ~50GB during processing
- Category 052-054: ~1GB each during processing

## Data Directory Structure

**Location**: Defined in `.env.shared` as `DATA_DIR_JUDICIAL` (default: `/Users/terry/Documents/opendata_judicial`)

```
opendata_judicial/
+-- meta.json                           # Root metadata (--categories)
+-- {category_id}/                      # e.g., 051, 052, 053, 054
    +-- meta.json                       # Category metadata (auto-saved during sync)
    +-- {dataset_id}/                   # e.g., 46391
        +-- {fileset_id}/               # e.g., 58452
            +-- {filename}.rar          # Downloaded archive (ZIP/7Z/RAR)
            +-- status.json             # Pipeline status tracking
            +-- extend/                 # Extracted content
                +-- {archive_name}/     # e.g., 199601
                    +-- {court_name}/   # e.g., 最高法院刑事
                        +-- {jid}.json          # Original judgment document
                        +-- {jid}_embed/        # Embedding output directory
                            +-- chunk_0.json    # Chunked text
                            +-- chunk_1.json
                            +-- embed_0.json    # Embedding vectors
                            +-- embed_1.json
```

### Key Files

| File | Description |
|------|-------------|
| `meta.json` (root) | Categories API response (`bun run src/index.ts sync judicial --categories`) |
| `meta.json` (category) | Resources API response (auto-saved during sync) |
| `status.json` | Pipeline progress (phases, counts, errors) |
| `{jid}.json` | Original judgment document (jfull, jdate, etc.) |
| `chunk_*.json` | Text chunks from CHUNK phase |
| `embed_*.json` | Embedding vectors from EMBED phase |
| `embeddings.jsonl.gz` | Compressed embeddings (ZIP phase output, in fileset dir) |

### Finding Filesets

```typescript
// Find filesets with fewest documents (for testing)
import { readdirSync, readFileSync } from 'fs';
import { join } from 'path';

const dataDir = '/Users/terry/Documents/opendata_judicial/051';
// Use recursive directory search to find status.json files
```

### Count Judgment Files (Excluding Chunk/Embed)

```bash
# Count original judgment files only
find /path/to/fileset/extend -type f -name "*.json" \
  ! -name "chunk_*.json" ! -name "embed_*.json" | wc -l
```

### JSON File Filtering

The `_is_in_embed_dir()` function filters out chunk and embed output files:
- Excludes files in `*_embed/` directories (e.g., `{jid}_embed/chunk_0.json`)
- Ensures only original judgment documents are processed
- Used during chunk phase to prevent reprocessing derived files
