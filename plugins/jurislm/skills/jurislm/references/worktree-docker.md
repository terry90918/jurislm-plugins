# Worktree Docker Configuration

Configure independent Docker containers for git worktrees to avoid conflicts.

## Core Principle

**Never stop other worktree or main project Docker containers.** Each worktree requires its own independent Docker environment with unique container names, ports, volumes, and networks.

## Quick Start

### Option 1: Copy Template (Recommended)

```bash
# Copy template
cp docker-compose.worktree.example.yml docker-compose.015.yml

# Replace all <SPEC> with spec number
sed -i '' 's/<SPEC>/015/g' docker-compose.015.yml

# Start services
docker compose -f docker-compose.015.yml up -d
```

### Option 2: Manual sed Command

```bash
# Alternative: Use sed to replace all <SPEC> placeholders
cp docker-compose.worktree.example.yml docker-compose.015.yml
sed -i '' 's/<SPEC>/015/g' docker-compose.015.yml
```

> Note: The legacy `generate-compose.sh` script has been removed. Use Option 1 (Copy Template) or the sed command above.

## Port Mapping

### Main Project vs Worktree

| Service | Main Port | Worktree Offset | Example (plan-a) |
|---------|-----------|-----------------|-------------------|
| PostgreSQL | 5432 | +1 | 5433 |

### Complete Port Allocation

| Worktree | DB Port |
|----------|---------|
| main | 5432 |
| plan-a | 5433 |
| plan-b | 5434 |

**Pattern for worktree N**:
- DB: `5432 + N`

## Naming Convention

Apply spec number suffix to all Docker resources:

| Resource | Main Project | Worktree (spec 014) |
|----------|--------------|---------------------|
| Container | `jurislm_db` | `jurislm_db_014` |
| Volume | `jurislm_data` | `jurislm_data_014` |
| Network | `jurislm_network` | `jurislm_network_014` |

## Docker Compose Template

```yaml
# docker-compose.014.yml
services:
  jurislm_db_014:
    build:
      context: ./jurislm_db
      dockerfile: Dockerfile
    image: jurislm_db_014
    container_name: jurislm_db_014
    ports:
      - "6432:5432"
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: jurislm_db
    volumes:
      - jurislm_data_014:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - jurislm_network_014

networks:
  jurislm_network_014:
    name: jurislm_network_014

volumes:
  jurislm_data_014:
```

## CLI Configuration

Update environment variables for worktree ports:

### .env.shared (worktree)

```bash
# Local database for worktree
DATABASE_URL=postgresql://postgres:postgres@localhost:6432/jurislm_db

# Shared database (same for all worktrees)
SHARED_DATABASE_URL=postgresql://postgres:postgres@localhost:5440/jurislm_shared_db

# Embedding provider (ollama = default, tei = backup)
EMBEDDING_PROVIDER=ollama
EMBEDDING_URL=http://localhost:11434
```

## Management Commands

### Start Worktree Services

```bash
docker compose -f docker-compose.014.yml up -d
```

### Stop Worktree Services

```bash
docker compose -f docker-compose.014.yml down
```

### Check Status

```bash
docker compose -f docker-compose.014.yml ps
```

### View Logs

```bash
docker compose -f docker-compose.014.yml logs -f jurislm_db_014
```

### Clean Up

```bash
# Stop and remove containers
docker compose -f docker-compose.014.yml down

# Also remove volumes (data loss!)
docker compose -f docker-compose.014.yml down -v
```

## Common Issues

### Port Already Allocated

Check running containers and processes:

```bash
docker ps --format "table {{.Names}}\t{{.Ports}}"
lsof -i :<port>
```

### Volume Name Conflicts

Docker Compose may reuse existing volumes. Always use unique volume names with spec suffix:

```yaml
volumes:
  jurislm_db_data_014:  # Not jurislm_db_data
```

### Network Isolation

Each worktree has its own network. Services communicate differently depending on context:

```yaml
# Container-to-container (within Docker network)
DATABASE_URL: postgresql://postgres:postgres@jurislm_db_014:5432/jurislm_db

# Host machine access (outside Docker network)
# Use: postgresql://postgres:postgres@localhost:6432/jurislm_db
```

> Note: Use service names (e.g., `jurislm_db_014`) for container-to-container communication. Use `localhost:<external_port>` when accessing from the host machine.

### Shared Database Access

To access shared database from worktree containers:

```yaml
environment:
  SHARED_DATABASE_URL: postgresql://postgres:postgres@host.docker.internal:5440/jurislm_shared_db
```

## Best Practices

1. **Never modify main project compose file** - Create separate file for worktree
2. **Use consistent naming** - Always append spec number suffix
3. **Verify ports before starting** - Avoid conflicts with running services
4. **Document OAuth changes** - Track added callback URLs
5. **Clean up after feature completion** - Remove unused containers and volumes

## Directory Structure

```
jurislm.worktrees/
├── main/                    # Main project
│   └── docker-compose.yml   # Main compose file (jurislm_db)
│
├── plan-a/                  # Worktree for spec 014
│   ├── docker-compose.014.yml
│   └── .env.shared          # Worktree-specific env
│
└── plan-b/                  # Worktree for spec 015
    ├── docker-compose.015.yml
    └── .env.shared          # Worktree-specific env
```
