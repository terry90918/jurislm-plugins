# Pipeline Architecture Reference

Two pipeline architectures handle data synchronization for judicial and law data.

## Architecture Comparison

| Feature | Judicial Pipeline | Law Pipeline |
|---------|------------------|-------------|
| Core Class | `SyncPipeline` + `PhaseExecutor` | `LawPipeline` |
| Design Pattern | IoC (Executor-per-phase) | Method-oriented |
| Processing Unit | Fileset (each fileset completes all 9 stages independently) | Pcode (CHUNK/EMBED per-pcode, others global) |
| Status Tracking | `StatusManager` + `status.json` (per-fileset) | `LawStatusManager` (per-pipeline) |
| Checkpoint Recovery | Automatic (status.json records each phase status) | Automatic (status.json tracks global + per-pcode) |
| Stages | 9: DOWNLOAD → UNZIP → PARSE → CHUNK → EMBED → ZIP → NAS → DB_UPLOAD → CLEANUP | 9: DOWNLOAD → UNZIP → PARSE → CHUNK → EMBED → ZIP → NAS → IMPORT → CLEANUP |

## Pipeline Stages (9-Stage)

### Judicial Pipeline

```
DOWNLOAD → UNZIP → PARSE → CHUNK → EMBED → ZIP → NAS → DB_UPLOAD → CLEANUP
```

| Stage | Executor Class | Per-Fileset | Description |
|-------|---------------|-------------|-------------|
| DOWNLOAD | `DownloadExecutor` | Yes | Fetch compressed filesets from Judicial Yuan API |
| UNZIP | `UnzipExecutor` | Yes | Extract files (auto-detect ZIP/7Z/RAR) |
| PARSE | `ParseExecutor` | Yes | PDF parsing (052-054: attorneys, justices, opinions) |
| CHUNK | `ChunkExecutor` | Yes | Metadata Context text splitting (no LLM) |
| EMBED | `EmbedExecutor` | Yes | Generate bge-m3 embeddings (1024 dim) |
| ZIP | `ZipExecutor` | Yes | Compress embeddings to JSONL.gz |
| NAS | `NasExecutor` | Yes | Upload to Synology NAS |
| DB_UPLOAD | `DbUploadExecutor` | Yes | Import to shared database (ON CONFLICT UPSERT) |
| CLEANUP | `CleanupExecutor` | Yes | Remove extend/ dir + embeddings.jsonl.gz |

### Law Pipeline

```
DOWNLOAD → UNZIP → PARSE → CHUNK → EMBED → ZIP → NAS → IMPORT → CLEANUP
```

| Stage | Method | Scope | Description |
|-------|--------|-------|-------------|
| DOWNLOAD | `executeGlobalPhase("DOWNLOAD")` | Global | Fetch ChLaw.zip + ChOrder.zip |
| UNZIP | `executeGlobalPhase("UNZIP")` | Global | Extract JSON files |
| PARSE | `executeGlobalPhase("PARSE")` | Global | Parse JSON, extract pcode, output source.json |
| CHUNK | Per-pcode loop | Per-pcode | Split articles with Metadata Context |
| EMBED | Per-pcode loop | Per-pcode | Generate bge-m3 embeddings |
| ZIP | `executeGlobalPhase("ZIP")` | Global | Compress to JSONL.gz |
| NAS | `executeGlobalPhase("NAS")` | Global | Upload to Synology NAS |
| IMPORT | `executeGlobalPhase("IMPORT")` | Global | Import to DB via LawImportWorker |
| CLEANUP | `executeGlobalPhase("CLEANUP")` | Global | Remove pcode dirs + ChLaw/ChOrder.json |

## Core Class Mapping

| Class | File Path | Responsibility |
|-------|-----------|----------------|
| `SyncPipeline` | `core/pipeline/sync-pipeline.ts` | Judicial Pipeline orchestrator |
| `PhaseExecutor` | `adapters/worker/base.ts` | Phase executor abstract class |
| `DbUploadExecutor` | `adapters/worker/db-upload.ts` | Judicial DB upload executor |
| `CleanupExecutor` | `adapters/worker/cleanup.ts` | Judicial cleanup executor |
| `LawPipeline` | `core/law-pipeline.ts` | Law Pipeline orchestrator |
| `StatusManager` | `core/status-manager.ts` | Judicial status.json management |
| `PipelineContext` | `core/context.ts` | Judicial execution environment |
| `FilesetContext` | `core/fileset-context.ts` | Judicial fileset context |

## Idempotency Strategy

Both pipelines are safe to re-execute (idempotent).

### Judicial (ON CONFLICT UPSERT)

All INSERTs have ON CONFLICT protection:

| Table | Unique Key | ON CONFLICT Strategy |
|-------|-----------|---------------------|
| `datasets` | dataset_id | DO NOTHING |
| `filesets` | fileset_id | DO UPDATE (desc, format) |
| `documents_051` | jid (PK) | DO UPDATE (jfull, file_path) |
| `documents_052` | inte_no (UK) | DO UPDATE (desc, reason, file_path) |
| `documents_053/054` | ruling_id (UK) | DO UPDATE (content, reason, file_path) |
| `document_embeddings_051` | (jid, model, chunk_strategy, chunk_index) | DO UPDATE (embedding, text) |
| `document_embeddings_052/053/054` | (document_id, model, chunk_index) | DO UPDATE (embedding, text) |

### Law (Transaction DELETE + INSERT)

Atomic operations within a single transaction:

| Table | Strategy |
|-------|----------|
| `laws` | ON CONFLICT (pcode) DO UPDATE |
| `law_articles` | DELETE + INSERT within transaction |
| `law_attachments` | DELETE + INSERT within transaction |
| `law_embeddings` | DELETE + INSERT within transaction |

## Cleanup Behavior

### Judicial Cleanup (Precise)

- Deletes: `extend/` directory (extracted content + chunks + embeddings)
- Deletes: `embeddings.jsonl.gz` (compressed output)
- Preserves: `status.json` (checkpoint tracking)
- Preserves: Archive files (`.rar`, `.zip`, `.7z`)

### Law Cleanup

- Deletes: All `{pcode}_embed_{strategy}/` directories
- Deletes: `ChLaw.json` (~200MB) and `ChOrder.json` (~500MB)
- Preserves: `ChLaw.zip` (~25MB) and `ChOrder.zip` (~111MB) for re-extraction
- Preserves: `status.json` and `embeddings_{strategy}.jsonl.gz`

## Pre-flight Checks

The pipeline performs automatic pre-flight checks before execution:

| Check | Condition | Required For |
|-------|-----------|-------------|
| Ollama connection | `curl localhost:11434` | CHUNK, EMBED phases |
| NAS connection | Synology API auth | NAS phase |
| Shared DB connection | PostgreSQL ping | DB_UPLOAD / IMPORT phase |

If a check fails, the pipeline exits with an error message before starting any work.
