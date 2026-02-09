# Shared Database Sync Troubleshooting

Common issues and solutions for jurislm_shared_db synchronization.

## Connection Issues

### Database Connection Failed

**Symptom**:
```
Error: Connection refused to localhost:5440
```

**Solution**:
```bash
# 1. Check if container is running
docker compose -f docker-compose.shared.yml ps

# 2. Start if not running
docker compose -f docker-compose.shared.yml up -d

# 3. Check container logs
docker compose -f docker-compose.shared.yml logs jurislm_shared_db

# 4. Verify port mapping
docker port jurislm_shared_db
```

### Ollama Connection Failed

**Symptom**:
```
Error: Cannot connect to Ollama at localhost:11434
```

**Solution**:
```bash
# 1. Check Ollama status
ollama list

# 2. Start Ollama if needed
ollama serve &

# 3. Verify bge-m3 model
ollama list | grep bge-m3

# 4. Pull model if missing
ollama pull bge-m3
```

### NAS Connection Failed

**Symptom**:
```
Error: SSH connection to NAS failed
```

**Solution**:
```bash
# 1. Test SSH connection manually
ssh -p $NAS_PORT $NAS_USERNAME@$NAS_HOST

# 2. Check SSH key authentication
ssh-add -l

# 3. Verify NAS service is running
ping $NAS_HOST

# 4. Check firewall rules on NAS
```

## Migration Issues

### Migration Version Mismatch

**Symptom**:
```
Error: Migration checksum mismatch for 001_initial.sql
```

**Solution**:
```bash
# 1. Check migration status
bun run src/index.ts db status --target shared --info

# 2. If development environment, reset database
bun run src/index.ts db reset --target shared

# 3. Re-apply migrations
bun run src/index.ts db migrate --target shared
```

### Table Already Exists

**Symptom**:
```
Error: relation "documents_051" already exists
```

**Solution**:
```bash
# Option 1: Skip to next migration
# Edit migrations table to mark as applied

# Option 2: Full reset (development only)
bun run src/index.ts db reset --target shared
```

## Sync Issues

### Judicial Sync Stuck

**Symptom**:
Sync hangs at EMBED phase for extended period.

**Solution**:
```bash
# 1. Check Ollama logs
journalctl -u ollama -f

# 2. Monitor embedding progress
# Look for "Processing embedding X/Y" in output

# 3. Restart with smaller batch
bun run src/index.ts sync judicial --category 051 --limit 100

# 4. Check system resources
htop  # Monitor CPU/Memory
```

### Download Phase Failed

**Symptom**:
```
Error: Failed to download from 司法院 API
```

**Solution**:
```bash
# 1. Check API availability
curl -I https://opendata.judicial.gov.tw

# 2. Retry with verbose output
bun run src/index.ts sync judicial --category 051 --info

# 3. Check network connectivity
ping opendata.judicial.gov.tw

# 4. Wait and retry (API rate limiting)
sleep 60 && bun run src/index.ts sync judicial --category 051
```

### Law Import Failed

**Symptom**:
```
Error: Duplicate key violation on laws table
```

**Solution**:
```bash
# 1. Check existing data
psql $SHARED_DATABASE_URL -c "SELECT pcode FROM laws WHERE pcode = 'A0000001';"

# 2. Force reprocess
bun run src/index.ts sync law --mode import --force

# 3. Or clear specific law data
psql $SHARED_DATABASE_URL -c "DELETE FROM laws WHERE pcode = 'A0000001';"
```

## Performance Issues

### Slow Embedding Generation

**Symptom**:
EMBED phase takes much longer than expected (< 3.4 embeddings/s).

**Solution**:
```bash
# 1. Check Ollama GPU usage
nvidia-smi  # If GPU available

# 2. Restart Ollama to clear memory
killall ollama && ollama serve &

# 3. Reduce concurrent load
# Run only one sync at a time

# 4. Consider using TEI (GPU optimized)
export EMBEDDING_PROVIDER=tei
bun run src/index.ts sync judicial --category 051
```

### Out of Memory

**Symptom**:
```
Error: JavaScript heap out of memory
```

**Solution**:
```bash
# 1. Increase Node.js memory limit
NODE_OPTIONS="--max-old-space-size=8192" bun run src/index.ts sync judicial

# 2. Process smaller batches
bun run src/index.ts sync judicial --category 051 --limit 500

# 3. Clear system cache
sync && echo 3 > /proc/sys/vm/drop_caches  # Linux only
```

### Disk Space Full

**Symptom**:
```
Error: No space left on device
```

**Solution**:
```bash
# 1. Check disk usage
df -h

# 2. Clean up temporary files
rm -rf jurislm_cli/data/judicial/*/raw/*.json
rm -rf jurislm_cli/data/law/raw/*.json

# 3. Clean Docker
docker system prune -a

# 4. Remove old embeddings
rm -rf jurislm_cli/data/judicial/*/embeddings/*.zip.old
```

## Data Integrity Issues

### Missing Embeddings

**Symptom**:
Documents exist but embeddings table is empty or incomplete.

**Solution**:
```bash
# 1. Check embedding counts
psql $SHARED_DATABASE_URL -c "
SELECT
  (SELECT COUNT(*) FROM documents_051) as docs,
  (SELECT COUNT(*) FROM document_embeddings_051) as embeddings;
"

# 2. Re-run embed phase only
bun run src/index.ts sync judicial --category 051 --mode embed

# 3. Force reprocess all
bun run src/index.ts sync judicial --category 051 --force
```

### Corrupted Data

**Symptom**:
Queries return unexpected results or errors.

**Solution**:
```bash
# 1. Verify data integrity
psql $SHARED_DATABASE_URL -c "
SELECT jid FROM documents_051 WHERE jfull IS NULL OR jfull = '';
"

# 2. Remove corrupted records
psql $SHARED_DATABASE_URL -c "
DELETE FROM documents_051 WHERE jfull IS NULL OR jfull = '';
"

# 3. Re-sync affected category
bun run src/index.ts sync judicial --category 051 --force
```

## Environment Issues

### Environment Variables Not Loaded

**Symptom**:
```
Error: SHARED_DATABASE_URL is not defined
```

**Solution**:
```bash
# 1. Check .env.shared exists
ls -la .env.shared

# 2. Copy from example if missing
cp .env.shared.example .env.shared

# 3. Verify variable is set
grep SHARED_DATABASE_URL .env.shared

# 4. Export manually for testing
export SHARED_DATABASE_URL=postgresql://postgres:postgres@localhost:5440/jurislm_shared_db
```

### Wrong Database Target

**Symptom**:
Sync writes to local database instead of shared.

**Solution**:
```bash
# 1. Always use --target shared flag
bun run src/index.ts db status --target shared
bun run src/index.ts db migrate --target shared

# 2. Verify connection string
echo $SHARED_DATABASE_URL

# 3. Check which database CLI connects to
bun run src/index.ts db status --info
```

## Recovery Procedures

### Full Reset and Resync

When issues are unrecoverable, perform full reset:

```bash
# 1. Stop all sync processes
pkill -f "sync judicial"
pkill -f "sync law"

# 2. Reset shared database
bun run src/index.ts db reset --target shared

# 3. Clean local data
rm -rf jurislm_cli/data/judicial/
rm -rf jurislm_cli/data/law/

# 4. Start fresh sync
bun run src/index.ts db migrate --target shared
bun run src/index.ts sync judicial
bun run src/index.ts sync law --mode import
```

### Partial Recovery

When only specific category needs resync:

```bash
# 1. Clear category data from database
psql $SHARED_DATABASE_URL -c "
DELETE FROM document_embeddings_051;
DELETE FROM documents_051;
"

# 2. Clear local files
rm -rf jurislm_cli/data/judicial/051/

# 3. Re-sync category
bun run src/index.ts sync judicial --category 051
```

## Monitoring

### Check Sync Progress

```bash
# Watch log output
bun run src/index.ts sync judicial --category 051 --info 2>&1 | tee sync.log

# Monitor in separate terminal
tail -f sync.log | grep -E "(Phase|Progress|Complete)"
```

### Database Health Check

```bash
# Check table sizes
psql $SHARED_DATABASE_URL -c "
SELECT
  schemaname,
  tablename,
  pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename)) as size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname || '.' || tablename) DESC;
"
```

### System Resource Monitoring

```bash
# CPU and Memory
htop

# Disk I/O
iotop

# Network
nethogs
```
