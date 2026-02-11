---
name: jurislm-taxonomy-generate
description: >-
  This skill should be used when the user asks to "build synonyms", "generate synonyms",
  "create legal synonyms", "taxonomy generate", "build taxonomy", "expand synonym database",
  "add synonyms to taxonomy", "populate synonym table", "legal term expansion",
  "生成同義詞", "建立同義詞", "擴充同義詞資料庫", "法律術語擴展",
  or mentions specific categories like "civil_debt synonyms", "criminal synonyms",
  "procedural synonyms", "labor synonyms". Claude Code generates Taiwan legal
  synonyms directly using its knowledge and inserts them into the legal_synonyms table.
version: 1.0.0
---

# Taxonomy Generate Skill

Generate Taiwan legal synonyms directly using Claude Code's knowledge and insert into the database.

## Overview

This skill enables Claude Code to generate Taiwan legal synonyms without calling external LLM APIs. Claude Code generates synonyms based on its legal knowledge and directly inserts them into `legal_synonyms` table.

**Target**: 2,600 synonym groups (11 categories x ~236 groups each)

## Workflow

### Step 1: Query Current Progress

```bash
docker exec jurislm_shared_db psql -U postgres -d jurislm_shared_db -c "
SELECT category, COUNT(DISTINCT canonical_term) as current_count,
  CASE category
    WHEN 'civil_debt' THEN 300 WHEN 'criminal' THEN 300
    WHEN 'procedural' THEN 300 WHEN 'administrative' THEN 300
    WHEN 'constitutional' THEN 200 WHEN 'labor' THEN 200
    WHEN 'ip' THEN 200 WHEN 'corporate' THEN 200
    WHEN 'tax' THEN 200 WHEN 'financial' THEN 200
    WHEN 'family' THEN 200
  END as target_count
FROM legal_synonyms GROUP BY category ORDER BY category;
"
```

### Step 2: Query Existing Terms

Avoid duplicates by checking existing terms first:

```bash
docker exec jurislm_shared_db psql -U postgres -d jurislm_shared_db -c "
SELECT DISTINCT canonical_term FROM legal_synonyms
WHERE category = '<CATEGORY>' ORDER BY canonical_term;
"
```

### Step 3: Generate Synonyms

Generate 10-20 synonym groups per interaction. Each group needs:
- `canonical_term`: Standard/primary term
- `synonyms`: 3-8 related terms (including canonical term)

**Quality Requirements**:
- Terms must be actual Taiwan legal terminology
- Include formal, colloquial terms, and abbreviations
- Avoid overly broad terms

See **`references/category-definitions.md`** for detailed category descriptions and sample synonym groups.

### Step 4: Insert into Database

**Important**: Use same `group_id` for all synonyms in a group. The unique constraint is `(category, canonical_term, synonym)`.

```bash
GROUP_ID=$(uuidgen)
docker exec jurislm_shared_db psql -U postgres -d jurislm_shared_db -c "
INSERT INTO legal_synonyms (group_id, canonical_term, synonym, category, source)
VALUES
  ('$GROUP_ID', '押金', '押金', 'civil_debt', 'claude-code-generated'),
  ('$GROUP_ID', '押金', '擔保金', 'civil_debt', 'claude-code-generated'),
  ('$GROUP_ID', '押金', '保證金', 'civil_debt', 'claude-code-generated')
ON CONFLICT (category, canonical_term, synonym) DO NOTHING;
"
```

**Note**: Always include the canonical_term itself as the first synonym in the group.

### Step 5: Verify Results

```bash
# Verify results
docker exec jurislm_shared_db psql -U postgres -d jurislm_shared_db -c "
SELECT category, COUNT(DISTINCT canonical_term) as groups, COUNT(*) as total
FROM legal_synonyms WHERE source = 'claude-code-generated'
GROUP BY category ORDER BY category;
"
```

## Category Quick Reference

| Category | Target | Focus Areas |
|----------|--------|-------------|
| civil_debt | 300 | Contracts, debt, property, security |
| criminal | 300 | Crimes, penalties, procedure |
| procedural | 300 | Litigation, parties, evidence |
| administrative | 300 | Government actions, appeals |
| constitutional | 200 | Rights, freedoms |
| labor | 200 | Employment, wages, disputes |
| ip | 200 | Copyright, patent, trademark |
| corporate | 200 | Companies, shareholders |
| tax | 200 | Tax types, collection |
| financial | 200 | Banking, insurance, securities |
| family | 200 | Marriage, custody, inheritance |

## Troubleshooting

- **Container not found**: Verify with `docker ps | grep shared_db`
- **UUID fails**: On macOS, `uuidgen` is built-in; on Linux, install `uuid-runtime`
- **Duplicate terms**: ON CONFLICT clause handles silently, no error
- **Connection refused**: Ensure `jurislm_shared_db` is accessible on port 5442 (46.225.58.202)

## Session Example

**User**: "Help me generate civil_debt synonyms"

**Claude Code Actions**:
1. Query current progress for civil_debt
2. Query existing terms to avoid duplicates
3. Generate 10-20 new synonym groups
4. Insert with proper UUIDs (one UUID per group)
5. Update meta table
6. Report results with counts

## Reference Files

- **`references/category-definitions.md`** - Detailed category descriptions and sample synonym groups for all 11 categories
