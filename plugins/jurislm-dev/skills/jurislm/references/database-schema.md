# Database Schema Reference

Complete database structure reference for JurisLM.

## Database Overview

| Metric | Value |
|--------|-------|
| Total Tables | 8 (local) + 19 (shared) = 27 |
| Local Migrations | 72 files |
| Shared Migrations | 15 files |
| Total Indexes | 30+ |
| Vector Dimension | 1024 (BAAI/bge-m3) |
| Database | PostgreSQL 18 + pgvector |

## Database Separation

JurisLM uses two separate PostgreSQL databases:

| Database | Port | Purpose |
|----------|------|---------|
| jurislm_db | 5433* | Local application data (auth, NextAuth, chat) |
| jurislm_shared_db | 5440 | Shared judicial/law/taxonomy data (19 tables) |

*Port varies by worktree (main: 5432, plan-a: 5433, plan-b: 5434)

**Shared Database Location**: Managed via `docker-compose.shared.yml`

## jurislm_db Tables (8 tables)

### Authentication & Users (3 tables)

| Table | Purpose | Key Fields |
|-------|---------|------------|
| users | End users (NextAuth.js) | id (TEXT/UUID), email, name, image |
| admin_users | Administrators (whitelist) | id, email, google_id, is_active |
| refresh_tokens | JWT refresh tokens | id, user_id, user_type, token, expires_at |

### NextAuth.js Standard Tables (3 tables)

| Table | Purpose | Key Fields |
|-------|---------|------------|
| accounts | OAuth account linking | provider, provider_account_id, user_id |
| sessions | Session management | session_token, user_id, expires |
| verification_tokens | Email verification | identifier, token, expires |

### Application (2 tables)

| Table | Purpose | Key Fields |
|-------|---------|------------|
| conversations | Chat conversations | id, user_id, title, model |
| messages | Chat messages | id, conversation_id, role, content |

### System (1 table - shared across both databases)

| Table | Purpose | Key Fields |
|-------|---------|------------|
| migrations | Migration tracking | id, migration_name, checksum, created_at |

## jurislm_shared_db Tables (19 tables)

### Judicial Open Data (11 tables)

| Table | Purpose | Key Fields |
|-------|---------|------------|
| categories | Category definitions (4 fixed) | category_no, category_name |
| datasets | Dataset metadata | dataset_id, category_no |
| filesets | Fileset tracking | fileset_id, dataset_id |
| documents_051 | Judgments (~21.7M source) | jid, fileset_id, jfull |
| documents_052 | Interpretations (~6,700 source) | jid, fileset_id, jfull |
| documents_053 | Decisions (~455 source) | jid, fileset_id, jfull |
| documents_054 | Rulings (~21 source) | jid, fileset_id, jfull |
| document_embeddings_051 | Judgment vectors | id, document_id, embedding |
| document_embeddings_052 | Interpretation vectors | id, document_id, embedding |
| document_embeddings_053 | Decision vectors | id, document_id, embedding |
| document_embeddings_054 | Ruling vectors | id, document_id, embedding |

### Law Data (4 tables)

| Table | Purpose | Key Fields |
|-------|---------|------------|
| laws | Law master table (includes foreword) | law_id, pcode, law_name, foreword |
| law_articles | Law articles | article_id, law_id, article_no, article_content |
| law_attachments | Law attachments | attachment_id, law_id, file_name, file_url |
| law_embeddings | Law vectors | embedding_id, law_id, chunk_index, embedding |

### Taxonomy (2 tables - in jurislm_shared_db)

| Table | Purpose | Key Fields |
|-------|---------|------------|
| legal_synonyms | Legal term synonym groups (2,600 groups, 9,990 records) | id, group_id, canonical_term, synonym, category |
| legal_concept_hierarchy | Broader/narrower concept DAG | id, child_term, parent_term, relation_type, category |

**Synonym Categories**: civil_debt, criminal, procedural, administrative, constitutional, labor, ip, corporate, tax, financial, family

### System (1 table)

| Table | Purpose | Key Fields |
|-------|---------|------------|
| migrations | Migration tracking | id, migration_name, checksum, created_at |

## Key Concepts

### pcode - Law Stable Identifier

The `pcode` field is extracted from MOJ URLs:

```
URL: https://law.moj.gov.tw/LawClass/LawAll.aspx?pcode=A0000001
pcode: A0000001
```

**Use pcode for**:
- Unique law identification (law names can change)
- Cross-reference between tables
- External API integration

### categories (Judicial Yuan)

The `categories` table contains 4 fixed document type categories from Judicial Yuan:

| category_id | Name |
|-------------|------|
| 051 | 裁判書 |
| 052 | 大法官解釋 |
| 053 | 憲法法庭判決 |
| 054 | 憲法法庭裁定 |

## Data Flow

### Judicial Open Data Flow

```
categories -> datasets -> filesets -> documents_*
                                        -> document_embeddings_*
```

### Law Data Flow

```
laws (pcode) -> law_articles (1..N)
            -> law_attachments (0..N)
            -> law_embeddings
```

## Naming Conventions

### Primary Keys

| Pattern | Example | Usage |
|---------|---------|-------|
| `id` | `id` | New tables |
| `{entity}_id` | `category_id` | Legacy tables |
| Business key | `jid`, `pcode` | Special cases |

### Foreign Keys

```
{table}_{column}_fkey
```
Example: `refresh_tokens_user_id_fkey`

### Indexes

| Type | Format | Example |
|------|--------|---------|
| General | `idx_{table}_{column}` | `idx_laws_pcode` |
| Composite | `idx_{table}_{col1}_{col2}` | `idx_docs_fileset_jid` |
| GIN | `idx_{table}_{column}_gin` | `idx_laws_articles_gin` |

### Constraints

| Type | Format | Example |
|------|--------|---------|
| CHECK | `{table}_{column}_check` | `chk_doc_embed_chunks` |
| UNIQUE | `{table}_{column}_key` | `laws_pcode_key` |

## ON DELETE Behavior

| Scenario | Behavior | Example |
|----------|----------|---------|
| Child depends on parent | CASCADE | filesets -> documents_* |
| Child cleanup | CASCADE | laws -> law_articles |
| Token cleanup | CASCADE | users -> refresh_tokens |

## Common Queries

### Find Law by pcode

```sql
SELECT * FROM laws WHERE pcode = 'A0000001';
```

### Check Abolished Laws

```sql
SELECT pcode, law_name, abolished_at
FROM laws WHERE is_abolished = true;
```

### Document Counts

```sql
SELECT 'documents_051' as tbl, COUNT(*) FROM documents_051
UNION ALL SELECT 'documents_052', COUNT(*) FROM documents_052
UNION ALL SELECT 'documents_053', COUNT(*) FROM documents_053
UNION ALL SELECT 'documents_054', COUNT(*) FROM documents_054;
```

### Query Expansion with Synonyms

```sql
-- Find all synonyms for a legal term
SELECT synonym FROM legal_synonyms
WHERE canonical_term = '押金';

-- Expand user query using synonyms
SELECT DISTINCT synonym FROM legal_synonyms
WHERE canonical_term IN (
  SELECT canonical_term FROM legal_synonyms WHERE synonym = '保證金'
);

-- Synonym statistics by category
SELECT category, COUNT(DISTINCT canonical_term) as groups, COUNT(*) as records
FROM legal_synonyms GROUP BY category ORDER BY groups DESC;
```

## Migration Strategy

**Total Migrations**: 72 files in `jurislm_db/migrations/`

### Naming Convention

```
YYYYMMDDHHmm_description.sql
```

### Key Migrations

| Migration | Description |
|-----------|-------------|
| 202511242245 | Initial complete schema (648 lines) |
| 202512021900 | Create users table |
| 202512021901 | Create admin_users table |
| 202512021902 | Create refresh_tokens table |
| 202601012317 | Remove sync progress tables (6 tables) |
| 202601020030 | Remove shared tables from jurislm_db (23 tables) |
| 202601020130 | Remove inheritance tables (4 tables) |

### Migration Commands

```bash
cd jurislm_cli
bun run src/index.ts db status              # View migration status
bun run src/index.ts db migrate             # Execute pending migrations
bun run src/index.ts db reset               # Reset database (data loss)

# Shared database
bun run src/index.ts db status --target shared
bun run src/index.ts db migrate --target shared
```

## Triggers

All tables use `update_updated_at_column()` trigger for automatic timestamp updates:

```sql
CREATE TRIGGER update_{table}_updated_at
BEFORE UPDATE ON {table}
FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```
