# Scheduled Sync Automation

Complete guide for automated judicial data synchronization using macOS launchd and Discord notifications.

## Overview

The scheduled sync system automates judicial data processing with:
- macOS launchd for scheduling (weekly/daily)
- Discord Webhook notifications for status updates
- Precheck system for service availability
- Environment variable security (whitelist)

## Architecture

```
launchd plist
    ↓
run-sync.sh (wrapper)
    ├── Load .env.shared (whitelist)
    ├── Precheck PostgreSQL
    ├── Precheck Ollama
    └── Execute: bun run src/index.ts sync judicial --category <cat>
         ├── Discord: Sync Start
         ├── 9-stage pipeline
         └── Discord: Sync Complete/Failed
```

## Schedule Configuration

### 051 Weekly Schedule (com.jurislm.sync-051.plist)

| Setting | Value | Notes |
|---------|-------|-------|
| Schedule | Sunday 02:00 | Least disruptive time |
| Timeout | 14400s (4h) | Buffer for large datasets |
| Category | 051 | 裁判書 (21M+ documents) |

### 052-054 Daily Schedule (com.jurislm.sync-daily.plist)

| Setting | Value | Notes |
|---------|-------|-------|
| Schedule | Daily 03:00 | After 051 window |
| Timeout | 3600s (1h) | Sufficient for ~1,600 docs |
| Categories | 052,053,054 | 大法官解釋 + 憲法法庭 |

## Installation

### Prerequisites

1. **Node.js/Bun** installed and in PATH
2. **PostgreSQL** running on port 5442 (Hetzner cloud)
3. **Ollama** running on port 11434
4. **Discord Webhook URL** (optional but recommended)

### Install Steps

```bash
cd jurislm_cli

# 1. Configure environment
cp ../.env.shared.example ../.env.shared
# Edit .env.shared with your values

# 2. Set file permissions (security requirement)
chmod 600 ../.env.shared

# 3. Install launchd schedules
./scripts/install-launchd.sh

# 4. Verify installation
launchctl list | grep jurislm
```

### Manual Trigger

```bash
# Trigger 051 sync immediately
launchctl start com.jurislm.sync-051

# Trigger daily sync immediately
launchctl start com.jurislm.sync-daily
```

### View Logs

```bash
# Standard output
tail -f ~/Library/Logs/jurislm/sync-051.log
tail -f ~/Library/Logs/jurislm/sync-daily.log

# Error output
tail -f ~/Library/Logs/jurislm/sync-051.error.log
tail -f ~/Library/Logs/jurislm/sync-daily.error.log

# Discord fallback log (when webhook fails)
cat ~/Library/Logs/jurislm/discord-fallback.log
```

### Uninstall

```bash
./scripts/uninstall-launchd.sh
```

## Discord Notifications

### Configuration

Set `DISCORD_WEBHOOK_URL` in `.env.shared`:

```bash
DISCORD_WEBHOOK_URL=https://discord.com/api/webhooks/{id}/{token}
```

**Webhook URL Format**: Must match `^https://discord.com/api/webhooks/\d+/[\w-]+$`

### Notification Types

#### Sync Start (Blue)

```json
{
  "title": "JurisLM 同步開始",
  "description": "開始同步分類 051 裁判書",
  "color": 0x3498db,
  "fields": [
    { "name": "分類", "value": "051 裁判書" },
    { "name": "預估時間", "value": "約 3 小時" }
  ]
}
```

#### Sync Complete (Green)

```json
{
  "title": "JurisLM 同步完成",
  "description": "分類 051 裁判書 同步成功",
  "color": 0x57f287,
  "fields": [
    { "name": "分類", "value": "051 裁判書" },
    { "name": "處理數量", "value": "1,130 筆" },
    { "name": "耗時", "value": "2 小時 15 分" },
    { "name": "NAS 狀態", "value": "已上傳" }
  ]
}
```

#### Sync Failed (Red)

```json
{
  "title": "JurisLM 同步失敗",
  "description": "分類 051 裁判書 同步過程發生錯誤",
  "color": 0xed4245,
  "fields": [
    { "name": "分類", "value": "051 裁判書" },
    { "name": "失敗階段", "value": "EMBED" },
    { "name": "錯誤訊息", "value": "Connection refused" },
    { "name": "建議操作", "value": "`jurislm sync judicial --category 051`" }
  ]
}
```

#### Precheck Failed (Red)

Sent by `run-sync.sh` when PostgreSQL or Ollama is unavailable:

```json
{
  "title": "JurisLM 排程預檢查失敗",
  "description": "分類 051 同步無法啟動",
  "color": 0xed4245,
  "fields": [
    { "name": "分類", "value": "051" },
    { "name": "錯誤", "value": "PostgreSQL 無法連線" },
    { "name": "建議", "value": "請確認服務狀態後手動執行同步" }
  ]
}
```

### Fallback Mechanism

When Discord notification fails (3 retries, 5s interval):

1. **Local file**: `~/Library/Logs/jurislm/discord-fallback.log`
2. **stderr**: Structured JSON output
3. **Never silent**: Follows Constitution Rule #6

## Security

### Environment Variable Whitelist

Only these variables are loaded from `.env.shared`:

```bash
DISCORD_WEBHOOK_URL
SHARED_DATABASE_URL
OLLAMA_BASE_URL
OLLAMA_MODEL
EMBEDDING_PROVIDER
CHUNK_CONTEXT_CONCURRENCY
CHUNK_FILE_CONCURRENCY
NAS_HOST
NAS_PORT
NAS_USERNAME
NAS_PASSWORD
NAS_SHARE_NAME
NAS_BASE_PATH

# DEPRECATED (2026-01-15): LLM_PROVIDER 已棄用，原因：改為 model 主導供應商選擇，避免混淆
```

### Security Checks

1. **File permissions**: `.env.shared` must be 600 or 400
2. **File owner**: Must match current user
3. **Value validation**: Rejects values containing:
   - Newlines (`\n`, `\r`)
   - Command substitution (`$()`, backticks)
   - Pipes, semicolons, ampersands (`|`, `;`, `&`)
   - Redirects (`>`, `<`)

### Category Validation

Only digits and commas allowed (regex: `^[0-9,]+$`)

```bash
# Valid
051
052,053,054

# Invalid (rejected)
051; rm -rf /
051 --extra-flag
```

## Troubleshooting

### Schedule Not Running

1. Check launchd status:
   ```bash
   launchctl list | grep jurislm
   ```

2. Check for errors:
   ```bash
   launchctl error jurislm com.jurislm.sync-051
   ```

3. Verify plist syntax:
   ```bash
   plutil -lint ~/Library/LaunchAgents/com.jurislm.sync-051.plist
   ```

### Discord Notifications Not Sending

1. Verify webhook URL format
2. Check fallback log:
   ```bash
   cat ~/Library/Logs/jurislm/discord-fallback.log
   ```
3. Test webhook manually:
   ```bash
   curl -X POST -H "Content-Type: application/json" \
     -d '{"content": "Test"}' \
     "$DISCORD_WEBHOOK_URL"
   ```

### Precheck Failures

**PostgreSQL**:
```bash
pg_isready -d "$SHARED_DATABASE_URL"
```

**Ollama**:
```bash
curl -s http://localhost:11434/api/tags
```

### PATH Issues

If `node` or `bun` not found:

1. Check `install-launchd.sh` output for detected paths
2. Verify PATH in installed plist:
   ```bash
   grep -A2 PATH ~/Library/LaunchAgents/com.jurislm.sync-051.plist
   ```

## Configuration Files Reference

### plist Template Variables

| Placeholder | Replaced With |
|-------------|---------------|
| `__SCRIPTS_DIR__` | Absolute path to `jurislm_cli/scripts/` |
| `__CLI_DIR__` | Absolute path to `jurislm_cli/` |
| `__LOG_DIR__` | `~/Library/Logs/jurislm` |
| `__EXTRA_PATH__` | Detected node/bun bin directories |

### run-sync.sh Flow

```
1. Validate category format (security)
2. Load .env.shared (whitelist)
3. Check Node.js availability
4. Check PostgreSQL connectivity
5. Check Ollama service
6. Execute CLI command
7. Log exit code
```

### Discord Notifier Features

| Feature | Implementation |
|---------|----------------|
| Retry | 3 attempts, 5s interval |
| Truncation | 1020 chars max per field |
| Fallback | Local file + stderr JSON |
| Webhook validation | Strict regex pattern |

## Performance Considerations

### 051 Processing Time

With metadata-context strategy:

| Dataset | Documents | Estimated Time |
|---------|-----------|----------------|
| Single fileset | ~1,130 | ~10 minutes |
| Full 051 | ~21M | < 3 hours |

**Bottleneck**: EMBED phase (~3.4 embeddings/s)

### Resource Usage

| Setting | Recommendation |
|---------|----------------|
| LowPriorityIO | Enabled (plist) |
| Memory | ~2GB during EMBED |
| CPU | Moderate (Ollama) |
| Network | NAS upload at end |
