# Shared Database Sync Workflow - Complete Reference

Detailed command reference for full synchronization of jurislm_shared_db.

## Complete Table List (19 Tables)

### System (1 table)
- `migrations` - Migration tracking

### Judicial (11 tables)
- `categories` - 4 fixed categories (051-054)
- `datasets` - Dataset metadata
- `filesets` - Fileset tracking
- `documents_051` - 裁判書 (~21M)
- `documents_052` - 大法官解釋 (~800)
- `documents_053` - 憲法法庭判決 (~300)
- `documents_054` - 憲法法庭裁定 (~500)
- `document_embeddings_051` - 裁判書向量
- `document_embeddings_052` - 解釋向量
- `document_embeddings_053` - 判決向量
- `document_embeddings_054` - 裁定向量

### Law (4 tables)
- `laws` - 法規主表 (~11,737)
- `law_articles` - 法規條文
- `law_attachments` - 法規附件
- `law_embeddings` - 法規向量

### Taxonomy (2 tables)
- `legal_synonyms` - 法律同義詞
- `legal_concept_hierarchy` - 概念階層 (DAG)

## Table Dependencies (Foreign Key Chain)

```
Judicial:
  categories (PK: category_no)
      ↓
  datasets (FK: category_no → categories)
      ↓
  filesets (FK: dataset_id → datasets)
      ↓
  documents_051/052/053/054 (FK: fileset_id → filesets)
      ↓
  document_embeddings_051/052/053/054 (FK: jid/document_id → documents_*)

Law:
  laws (PK: law_id)
      ↓
  law_articles (FK: law_id → laws)
  law_attachments (FK: law_id → laws)
  law_embeddings (FK: law_id → laws)

Taxonomy:
  legal_synonyms (independent)
  legal_concept_hierarchy (independent)
```

**Important**: Foreign key constraints enforce this order. Parent tables must be populated before child tables.

## Prerequisites Verification

### 1. Docker Services

Start shared services:

```bash
# Start services
docker compose -f docker-compose.shared.yml up -d

# Verify running
docker compose -f docker-compose.shared.yml ps
# Expected: jurislm_shared_db (Port 5440), jurislm-tei-shared (Port 8090)

# Check logs if issues
docker compose -f docker-compose.shared.yml logs jurislm_shared_db
```

### 2. Ollama Embedding Service

Verify Ollama is running with bge-m3 model:

```bash
# Check Ollama status
curl -s http://localhost:11434/api/tags | jq '.models[].name'

# Pull bge-m3 if needed
ollama pull bge-m3

# Test embedding
curl -s http://localhost:11434/api/embeddings -d '{
  "model": "bge-m3",
  "prompt": "test"
}' | jq '.embedding | length'
# Expected: 1024
```

### 3. Environment Variables

Required variables in `.env.shared`:

```bash
# Database
SHARED_DATABASE_URL=postgresql://postgres:postgres@localhost:5440/jurislm_shared_db

# Embedding
OLLAMA_BASE_URL=http://localhost:11434
EMBEDDING_PROVIDER=ollama

# NAS Upload (Synology)
NAS_HOST=your-nas-host
NAS_PORT=22
NAS_USERNAME=jurislm
NAS_PASSWORD=your-password
NAS_BASE_PATH=/volume1/jurislm
```

### 4. Database Connection Test

```bash
cd jurislm_cli

# Test connection
bun run src/index.ts db status --target shared

# Expected output:
# Database: jurislm_shared_db (shared)
# Status: Connected
# Migrations: X applied, Y pending
```

## Database Migration Commands

### Check Migration Status

```bash
# Basic status
bun run src/index.ts db status --target shared

# Detailed status with table info
bun run src/index.ts db status --target shared --info
```

### Apply Migrations

```bash
# Apply pending migrations
bun run src/index.ts db migrate --target shared

# With verbose output
bun run src/index.ts db migrate --target shared --info
```

### Reset Database (Destructive)

```bash
# Full reset - deletes all data
bun run src/index.ts db reset --target shared

# Skip Docker image rebuild
bun run src/index.ts db reset --target shared --skip-build
```

## Judicial Sync Commands

### View Available Categories

```bash
# List all categories with metadata
bun run src/index.ts sync judicial --categories
```

Output shows:
- Category ID (051-054)
- Category name
- Dataset count
- Total documents

### Sync All Categories

```bash
# Sync all 4 categories
bun run src/index.ts sync judicial

# With verbose output
bun run src/index.ts sync judicial --info
```

### Sync Specific Categories

```bash
# Single category
bun run src/index.ts sync judicial --category 051

# Multiple categories
bun run src/index.ts sync judicial --category 051,052

# All constitutional categories
bun run src/index.ts sync judicial --category 052,053,054
```

### Sync Options

```bash
# Limit documents (for testing)
bun run src/index.ts sync judicial --category 051 --limit 10

# Run specific phase only
bun run src/index.ts sync judicial --category 051 --mode embed
bun run src/index.ts sync judicial --category 051 --mode chunk

# Import to database only (skip pipeline, read from local files)
bun run src/index.ts sync judicial --category 051 --mode import

# Import specific dataset/fileset
bun run src/index.ts sync judicial --category 051 --mode import --dataset 46429 --fileset 58490

# Force reprocess completed items
bun run src/index.ts sync judicial --category 051 --force

# Verbose output
bun run src/index.ts sync judicial --category 051 --info
```

### Pipeline Phases

| Phase | Description | Output |
|-------|-------------|--------|
| DOWNLOAD | Fetch JSON from 司法院 API | `data/judicial/{category}/raw/*.json` |
| UNZIP | Extract compressed files | `data/judicial/{category}/extracted/` |
| PARSE | Parse JSON structure | In-memory document objects |
| CHUNK | Split text with metadata context | Chunked documents in `*_embed/chunk_*.json` |
| EMBED | Generate bge-m3 embeddings | 1024-dim vectors stored in `chunk_*.json` |
| ZIP | Compress embeddings | `data/judicial/{category}/embeddings/*.zip` |
| NAS | Upload to Synology NAS | Remote backup |

### Import Mode (--mode import)

Import mode reads existing local files and inserts into database:

| Step | Source | Target Table |
|------|--------|--------------|
| 1. Datasets | `meta.json` → `data[].datasetId` | `datasets` |
| 2. Filesets | `meta.json` → `data[].filesets[]` | `filesets` |
| 3. Documents | `extend/{period}/{court}/{JID}.json` | `documents_051/052/053/054` |
| 4. Embeddings | `{JID}_embed/chunk_*.json` | `document_embeddings_051/052/053/054` |

**FK Dependency Chain**:
```
categories (Migration) → datasets → filesets → documents_* → document_embeddings_*
```

**Data Requirements**:
- `meta.json` must exist in category directory
- Documents must be in `extend/` subdirectories
- Embeddings must have `embedding` field in `chunk_*.json` (from EMBED phase)

## Law Sync Commands

### Full Law Sync

```bash
# Sync all laws (no DB import)
bun run src/index.ts sync law

# Sync with database import
bun run src/index.ts sync law --mode import
```

### Sync Specific Law

```bash
# By pcode
bun run src/index.ts sync law --pcode A0000001

# Multiple pcodes
bun run src/index.ts sync law --pcode A0000001,B0000001
```

### Law Sync Options

```bash
# Run specific phase
bun run src/index.ts sync law --mode embed
bun run src/index.ts sync law --mode import    # DB import only

# Force reprocess
bun run src/index.ts sync law --force

# Verbose output
bun run src/index.ts sync law --info
```

### Law Pipeline Phases

| Phase | Description | Output |
|-------|-------------|--------|
| DOWNLOAD | Fetch from 法務部 API | `data/law/raw/*.json` |
| PARSE | Parse law structure | Law objects with articles |
| CHUNK | Split articles | Chunked content |
| EMBED | Generate embeddings | 1024-dim vectors |
| ZIP | Compress results | `data/law/embeddings/*.zip` |
| NAS | Upload to NAS | Remote backup |
| IMPORT | Insert to database | laws, law_articles, law_embeddings tables |

## Taxonomy Commands

### Build Synonyms

```bash
# Build with Claude Haiku 4.5 Batch API (default)
bun run src/index.ts taxonomy build

# Use Ollama (local, no API cost)
bun run src/index.ts taxonomy build --provider ollama

# Verbose output
bun run src/index.ts taxonomy build --info
```

### Query Synonyms

```bash
# Lookup term
bun run src/index.ts taxonomy lookup 押金

# Export all synonyms
bun run src/index.ts taxonomy export synonyms.json
```

### Statistics

```bash
# View taxonomy stats
bun run src/index.ts taxonomy stats

# Detailed stats
bun run src/index.ts taxonomy stats --info
```

## Complete Sync Script

Execute all sync steps in order:

```bash
#!/bin/bash
set -e

cd jurislm_cli

echo "=== Step 1: Database Migration ==="
bun run src/index.ts db migrate --target shared

echo "=== Step 2: Judicial Sync (051-054) ==="
bun run src/index.ts sync judicial --info

echo "=== Step 3: Law Sync with Import ==="
bun run src/index.ts sync law --mode import --info

echo "=== Step 4: Taxonomy Build ==="
bun run src/index.ts taxonomy build --info

echo "=== Sync Complete ==="
bun run src/index.ts db status --target shared --info
```

## Verification Queries

### Complete 19-Table Count (Single Query)

```sql
-- Verify ALL 19 tables have data
SELECT 'migrations' as table_name, COUNT(*) as count FROM migrations
UNION ALL SELECT 'categories', COUNT(*) FROM categories
UNION ALL SELECT 'datasets', COUNT(*) FROM datasets
UNION ALL SELECT 'filesets', COUNT(*) FROM filesets
UNION ALL SELECT 'documents_051', COUNT(*) FROM documents_051
UNION ALL SELECT 'documents_052', COUNT(*) FROM documents_052
UNION ALL SELECT 'documents_053', COUNT(*) FROM documents_053
UNION ALL SELECT 'documents_054', COUNT(*) FROM documents_054
UNION ALL SELECT 'document_embeddings_051', COUNT(*) FROM document_embeddings_051
UNION ALL SELECT 'document_embeddings_052', COUNT(*) FROM document_embeddings_052
UNION ALL SELECT 'document_embeddings_053', COUNT(*) FROM document_embeddings_053
UNION ALL SELECT 'document_embeddings_054', COUNT(*) FROM document_embeddings_054
UNION ALL SELECT 'laws', COUNT(*) FROM laws
UNION ALL SELECT 'law_articles', COUNT(*) FROM law_articles
UNION ALL SELECT 'law_attachments', COUNT(*) FROM law_attachments
UNION ALL SELECT 'law_embeddings', COUNT(*) FROM law_embeddings
UNION ALL SELECT 'legal_synonyms', COUNT(*) FROM legal_synonyms
UNION ALL SELECT 'legal_concept_hierarchy', COUNT(*) FROM legal_concept_hierarchy
ORDER BY table_name;
```

### Expected Counts (Fully Synced)

| Table | Expected Count | Notes |
|-------|----------------|-------|
| migrations | ~12 | Migration files applied |
| categories | 4 | Fixed (051-054) |
| datasets | ~100+ | Varies by API |
| filesets | ~1000+ | Varies by API |
| documents_051 | ~21,000,000 | 裁判書 |
| documents_052 | ~800 | 大法官解釋 |
| documents_053 | ~300 | 憲法法庭判決 |
| documents_054 | ~500 | 憲法法庭裁定 |
| document_embeddings_051 | > documents_051 | Multiple chunks per doc |
| document_embeddings_052 | > documents_052 | Multiple chunks per doc |
| document_embeddings_053 | > documents_053 | Multiple chunks per doc |
| document_embeddings_054 | > documents_054 | Multiple chunks per doc |
| laws | ~11,737 | 法律 + 命令 |
| law_articles | > laws | Multiple articles per law |
| law_attachments | Variable | Not all laws have attachments |
| law_embeddings | > laws | Multiple chunks per law |
| legal_synonyms | ~2,600 groups | Target: 9,990 records |
| legal_concept_hierarchy | Variable | DAG relationships |

### Data Integrity Checks

```sql
-- 1. Check for documents without embeddings (051)
SELECT COUNT(*) as missing_embeddings
FROM documents_051 d
LEFT JOIN document_embeddings_051 e ON d.jid = e.jid
WHERE e.jid IS NULL;

-- 2. Check for laws without embeddings
SELECT COUNT(*) as missing_embeddings
FROM laws l
LEFT JOIN law_embeddings e ON l.law_id = e.law_id
WHERE e.law_id IS NULL;

-- 3. Check for laws without articles
SELECT COUNT(*) as missing_articles
FROM laws l
LEFT JOIN law_articles a ON l.law_id = a.law_id
WHERE a.law_id IS NULL;

-- 4. Check synonym coverage by category
SELECT category, COUNT(DISTINCT group_id) as groups, COUNT(*) as records
FROM legal_synonyms
GROUP BY category
ORDER BY groups DESC;
```

### Quick Health Check

```sql
-- Summary view: tables with zero records (potential issues)
SELECT table_name, count
FROM (
  SELECT 'categories' as table_name, COUNT(*) as count FROM categories
  UNION ALL SELECT 'datasets', COUNT(*) FROM datasets
  UNION ALL SELECT 'filesets', COUNT(*) FROM filesets
  UNION ALL SELECT 'documents_051', COUNT(*) FROM documents_051
  UNION ALL SELECT 'laws', COUNT(*) FROM laws
  UNION ALL SELECT 'legal_synonyms', COUNT(*) FROM legal_synonyms
) t
WHERE count = 0;
-- Empty result = all critical tables have data
```

## Incremental Sync

For daily/weekly updates (not full sync):

```bash
# Sync only new judicial documents
bun run src/index.ts sync judicial --category 051

# Sync only new laws
bun run src/index.ts sync law --mode import

# The pipeline automatically skips already-processed items
```

## Performance Tuning

### Parallel Processing

Categories can be synced in parallel (separate terminal sessions):

```bash
# Terminal 1
bun run src/index.ts sync judicial --category 051

# Terminal 2
bun run src/index.ts sync judicial --category 052,053,054

# Terminal 3
bun run src/index.ts sync law --mode import
```

### Memory Optimization

For large datasets, use `--limit` to process in batches:

```bash
# Process 1000 documents at a time
bun run src/index.ts sync judicial --category 051 --limit 1000
```

### Disk Space

Monitor disk usage during sync:

```bash
# Check data directory size
du -sh jurislm_cli/data/

# Clean up old files
rm -rf jurislm_cli/data/judicial/*/raw/*.json
rm -rf jurislm_cli/data/law/raw/*.json
```
