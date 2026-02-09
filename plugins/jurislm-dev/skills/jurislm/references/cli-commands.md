# CLI Commands Reference

Complete reference for `jurislm` CLI commands.

## Command Structure

```bash
cd jurislm_cli
bun run src/index.ts [OPTIONS] COMMAND [SUBCOMMAND] [ARGS]
```

**Note**: Execute from `jurislm_cli` directory. Alternatively, from root: `bun run jurislm_cli/src/index.ts <command>`

## Turborepo Integration

The project uses Turborepo for unified task management:

```bash
# From root directory
bun run lint          # ESLint (all projects)
bun run test          # Tests (all projects)
bun run typecheck     # TypeScript check (all projects)

# Filter specific project
bunx turbo lint --filter=jurislm-cli
bunx turbo test --filter=jurislm-app
```

## Database Commands (db)

### db status

Display local migration status.

```bash
bun run src/index.ts db status
bun run src/index.ts db status --info       # Verbose output
bun run src/index.ts db status --target shared  # Shared database
```

Output shows:
- Applied migrations count
- Pending migrations list
- Last migration timestamp

### db migrate

Execute pending migrations.

```bash
bun run src/index.ts db migrate              # Execute all pending
bun run src/index.ts db migrate --info       # Detailed output
bun run src/index.ts db migrate --target shared  # Shared database
```

**Behavior**:
- Preserves existing data
- Executes migrations in timestamp order
- Records execution in `migrations` table

### db reset

Reset database completely.

```bash
bun run src/index.ts db reset                # Full reset
bun run src/index.ts db reset --skip-build   # Skip Docker rebuild
bun run src/index.ts db reset --target shared  # Reset shared database
```

**Warning**: Destroys all data. Use only in development.

**Operations**:
1. Stop jurislm_db container
2. Remove Docker volume
3. Rebuild container
4. Execute all migrations

### db push

Push migrations to remote environment.

```bash
bun run src/index.ts db push staging         # Push to staging
bun run src/index.ts db push production      # Push to production
bun run src/index.ts db push staging -f      # Skip confirmation
bun run src/index.ts db push staging --target shared  # Shared database
```

**Requirements**:
- Remote credentials in `jurislm_db/.env.staging` or `.env.production`

### db pull

Pull migrations from remote environment.

```bash
bun run src/index.ts db pull staging         # Pull from staging
bun run src/index.ts db pull production      # Pull from production
bun run src/index.ts db pull staging -f      # Skip confirmation
```

## Sync Commands (sync)

### sync judicial

Synchronize judicial documents from Taiwan Judicial Yuan.

```bash
# Sync all categories (default behavior)
bun run src/index.ts sync judicial

# Sync specific categories
bun run src/index.ts sync judicial --category 051
bun run src/index.ts sync judicial --category 051,052,053

# Execute only embed phase
bun run src/index.ts sync judicial --category 051 --mode embed

# Limit number of judgments to process
bun run src/index.ts sync judicial --category 051 --limit 10

# View all categories
bun run src/index.ts sync judicial --categories

# Verbose output
bun run src/index.ts sync judicial --category 051 --info
```

**Options**:

| Option | Description | Default |
|--------|-------------|---------|
| `--category <list>` | Comma-separated category IDs (051,052,053,054) | all |
| `--mode {phase}` | Execute only specified phase (download/unzip/parse/chunk/embed/zip/nas) | all |
| `--strategy {name}` | Chunk strategy (metadata-context only) | metadata-context |
| `--limit N` | Limit number of **judgments** to process (cross-fileset) | unlimited |
| `--dataset ID` | Process only specified dataset ID | - |
| `--fileset ID` | Process only specified fileset ID (requires --dataset) | - |
| `--force` | Force reprocess completed filesets | false |
| `--info` | Verbose output | false |
| `--categories` | Download and display all categories | false |

**Chunk Strategy** (Metadata Context):
- Only strategy: `metadata-context`
- Uses structured JSON metadata for context prefix (no LLM calls)
- Extracts: JID, JYEAR, JCASE, JTITLE, court_name
- Performance: ~0.001 sec/chunk (10,000x faster than previous LLM-based strategy)

**Breaking Change**: `--strategy contextual` is no longer supported.

**Category Codes**:

| Code | Name | Description |
|------|------|-------------|
| 051 | 裁判書 | 法院判決 |
| 052 | 大法官解釋 | 大法官憲法解釋 |
| 053 | 憲法法庭判決 | 憲法法庭判決 |
| 054 | 憲法法庭裁定 | 憲法法庭裁定 |

### sync law

Synchronize law data from Ministry of Justice (7-stage pipeline).

```bash
# Full sync (all 11,737 laws/orders)
bun run src/index.ts sync law

# Sync specific pcode(s)
bun run src/index.ts sync law --pcode A0000001
bun run src/index.ts sync law --pcode A0000001,A0000002

# Execute specific phase only
bun run src/index.ts sync law --mode download    # Download ChLaw.zip + ChOrder.zip
bun run src/index.ts sync law --mode embed       # Generate embeddings only
bun run src/index.ts sync law --mode import      # Import to database only

# Limit processing
bun run src/index.ts sync law --limit 10

# Force reprocess (ignore completed markers)
bun run src/index.ts sync law --force

# Sync and import to database
bun run src/index.ts sync law --import

# Verbose output
bun run src/index.ts sync law --info
```

**Options**:

| Option | Description | Default |
|--------|-------------|---------|
| `--pcode <list>` | Comma-separated pcode list (filters at PARSE phase) | all |
| `--mode {phase}` | Execute only specified phase (download/unzip/parse/chunk/embed/zip/nas/import) | all |
| `--strategy {name}` | Chunk strategy (metadata-context only) | metadata-context |
| `--limit N` | Limit number of laws to process | unlimited |
| `--force` | Force reprocess completed laws | false |
| `--import` | Import to database after embedding | false |
| `--info` | Verbose output | false |

**Option Conflicts**:
- `--pcode` + `--limit` cannot be used together

**Pipeline Stages**:
1. **DOWNLOAD** - Fetch ChLaw.zip + ChOrder.zip from MOJ API
2. **UNZIP** - Extract JSON files
3. **PARSE** - Parse JSON, extract pcode, merge article content, output source.json
4. **CHUNK** - Text splitting with Metadata Context (no LLM calls)
5. **EMBED** - Generate embeddings (Ollama bge-m3, 1024 dim)
6. **ZIP** - Compress output files
7. **NAS** - Upload to Synology NAS
8. **IMPORT** - Import to database (laws, law_articles, law_attachments, law_embeddings)

## Taxonomy Commands (taxonomy)

### taxonomy build

Generate, review, and import synonyms for all categories in one command.

```bash
# Build all categories (預設使用 Claude Haiku 4.5 + Batch API)
bun run src/index.ts taxonomy build

# 使用 Ollama（本地模型）
bun run src/index.ts taxonomy build --provider ollama

# 詳細輸出
bun run src/index.ts taxonomy build --info
```

**Options**:

| Option | Description | Default |
|--------|-------------|---------|
| `--provider <name>` | LLM provider: anthropic (Batch API, 50% 成本節省) 或 ollama | anthropic |
| `--info` | Show detailed output | false |

**Batch API 特點**:
- 預設使用 Claude Haiku 4.5（`claude-haiku-4-5-20251001`）
- 50% 成本節省（相比標準 API 呼叫）
- 無額外儲存費用（不同於 Voyage AI）
- 審核階段批次處理（減少 API 呼叫次數）

**Smart Resume**: Automatically queries existing data and only builds remaining items. Counts ALL synonyms (including manual imports) toward each category's target.

### taxonomy lookup

Query synonyms for a term.

```bash
bun run src/index.ts taxonomy lookup 押金
bun run src/index.ts taxonomy lookup 違約金 --info
```

### taxonomy stats

Display synonym statistics.

```bash
bun run src/index.ts taxonomy stats
bun run src/index.ts taxonomy stats --info   # Detailed breakdown
```

### taxonomy import / export

```bash
# Import from JSON files
bun run src/index.ts taxonomy import data/synonyms/*.json

# Export to JSON file
bun run src/index.ts taxonomy export output/synonyms.json
```

## Evaluate Commands (evaluate)

### evaluate embedding-similarity

Test embedding model's ability to capture semantic relationships.

```bash
bun run src/index.ts evaluate embedding-similarity
```

**Output**: Cosine distances for synonym pairs. Lower values = better semantic understanding.

### evaluate baseline

Run baseline vector search test without synonym expansion.

```bash
# Default settings
bun run src/index.ts evaluate baseline

# Custom parameters
bun run src/index.ts evaluate baseline -k 5 -s 0.7
```

**Options**:

| Option | Description | Default |
|--------|-------------|---------|
| `-k, --top-k <num>` | Number of results to retrieve | 5 |
| `-s, --min-score <num>` | Minimum similarity score | 0.65 |

**Output**: Recall@K metrics and list of failed queries for analysis.

### evaluate baseline-expanded

Run baseline test with synonym expansion enabled.

```bash
# Default settings
bun run src/index.ts evaluate baseline-expanded

# Custom expansion limit
bun run src/index.ts evaluate baseline-expanded -e 3

# Full options
bun run src/index.ts evaluate baseline-expanded -k 5 -s 0.7 -e 5
```

**Options**:

| Option | Description | Default |
|--------|-------------|---------|
| `-k, --top-k <num>` | Number of results to retrieve | 5 |
| `-s, --min-score <num>` | Minimum similarity score | 0.65 |
| `-e, --max-expansions <num>` | Maximum synonym expansions per term | 5 |

**Output**: Recall@K with expansion, comparison to baseline, expansion gain percentage.

## Environment Variables

### Required

```bash
# Database
DATABASE_URL=postgresql://postgres:postgres@localhost:5433/jurislm_db

# Shared database
SHARED_DATABASE_URL=postgresql://postgres:postgres@localhost:5440/jurislm_shared_db

# Judicial API
JUDICIAL_USERNAME=your_username
JUDICIAL_PASSWORD=your_password

# Embedding Provider (ollama = primary, tei = backup)
EMBEDDING_PROVIDER=ollama           # Options: ollama | tei
# Ollama default URL: http://localhost:11434
# TEI default URL: http://localhost:8090
```

### Optional

```bash
# NAS (Synology)
SYNOLOGY_BASE_URL=https://your-nas.synology.me:5001
SYNOLOGY_ACCOUNT=your-account
SYNOLOGY_PASSWORD=your-password

# Embedding URL (optional, has sensible defaults)
# Ollama default: http://localhost:11434
# TEI default: http://localhost:8090
EMBEDDING_URL=http://localhost:11434
```

## Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | General error |
| 2 | Invalid arguments |
| 3 | Database connection error |
| 4 | Service unavailable |

## CLI Architecture

```
jurislm_cli/
├── src/
│   ├── index.ts             # Entry point
│   ├── config.ts            # Configuration (Zod schema, lazy loaded)
│   ├── commands/            # Command definitions (lazy loading)
│   │   ├── db.ts            # Database commands
│   │   ├── sync.ts          # Sync commands
│   │   ├── taxonomy.ts      # Taxonomy commands
│   │   ├── evaluate.ts      # Evaluation commands
│   │   ├── impl/            # Command implementations (heavy dependencies)
│   │   │   ├── db.impl.ts
│   │   │   ├── sync.impl.ts
│   │   │   ├── taxonomy.impl.ts
│   │   │   └── evaluate.impl.ts
│   │   └── utils/
│   │       └── lazy-import.ts  # Lazy import utilities
│   ├── core/
│   │   ├── context.ts       # Execution context
│   │   ├── phase.ts         # Phase definitions
│   │   ├── pipeline.ts      # Pipeline orchestrator
│   │   └── embedding-providers/  # TEI/Ollama providers
│   └── adapters/
│       ├── db/              # Database adapters
│       └── worker/          # Worker adapters (7 phases)
└── tests/
    ├── unit/                # Unit tests
    └── integration/         # Integration tests
```

**Lazy Loading Architecture**: Commands are split into definitions (lightweight) and implementations (heavy dependencies). This reduces `--help` startup time from ~1800ms to ~30ms (**98% improvement**).

## Common Workflows

### Initial Setup

```bash
# 1. Start services
docker compose up -d

# 2. Check migration status
cd jurislm_cli
bun run src/index.ts db status

# 3. Execute migrations
bun run src/index.ts db migrate

# 4. Verify tables
docker exec jurislm_db psql -U postgres -d jurislm_db -c "\dt"
```

### Full Data Sync

```bash
# 1. Start shared services
docker compose -f docker-compose.shared.yml up -d

# 2. Sync judicial data
cd jurislm_cli
bun run src/index.ts sync judicial

# 3. Sync law data
bun run src/index.ts sync law

# 4. Verify counts
docker exec jurislm_shared_db psql -U postgres -d jurislm_shared_db -c "
SELECT 'documents_051' as tbl, COUNT(*) FROM documents_051
UNION ALL SELECT 'laws', COUNT(*) FROM laws;
"
```
