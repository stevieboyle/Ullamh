# [PROJECT_NAME] — Operations Runbook
**Version:** 1.0  
**Last updated:** [DATE]

> **Usage:** Keep this document up to date as the deployment topology changes. It is the first document an operator should read when something goes wrong. Every new deployment environment should have its own values filled in for the variable references below.

---

## 1. Deployment

### 1.1 Production Deployment

```bash
# Pull latest code
git pull origin main

# Build and restart the stack
docker compose down
docker compose up --build -d

# Verify services are up
docker compose ps
docker compose logs [api-service] --tail 50
```

**Post-deploy smoke check:**
```bash
curl -s http://[HOST]:[PORT]/api/health
# Expected: {"status": "ok"}
```

### 1.2 Environment Variables

All runtime configuration is in `.env`. Before deploying to a new environment:

```bash
cp .env.example .env
# Edit .env — review every variable before first boot
```

Critical production overrides:

| Variable | Dev default | Production value |
|---|---|---|
| `DATABASE_URL` | [local path] | [absolute container path] |
| `CORS_ORIGINS` | `["http://localhost:[PORT]"]` | `["https://[your-domain.com]"]` |
| `[SECRET_KEY_VAR]` | _(unset / dummy)_ | [Strong random secret] |
| `DEBUG` | `false` | `false` (NEVER `true` in production) |

### 1.3 Database Migrations

Run after every deployment that includes schema changes:

```bash
docker compose exec [api-service] [migration-tool] upgrade head
# e.g. alembic upgrade head
```

### 1.4 Rollback

```bash
# Rollback one schema version
docker compose exec [api-service] [migration-tool] downgrade -1

# Rollback to a specific revision
docker compose exec [api-service] [migration-tool] downgrade [revision-id]
```

---

## 2. Backup & Recovery

### 2.1 Database Backup

```bash
# Option A: Offline copy (stack down)
docker compose down
cp [db-file] [db-file].backup-$(date +%Y%m%d-%H%M%S)
docker compose up -d

# Option B: Online backup (database-specific command)
[db-backup-command]
```

### 2.2 File / Asset Backup

```bash
tar -czf [assets]-backup-$(date +%Y%m%d).tar.gz [assets-directory]/
```

### 2.3 Restoration

```bash
docker compose down
cp [db-file].backup-[timestamp] [db-file]
tar -xzf [assets]-backup-[date].tar.gz
docker compose up -d
```

---

## 3. Monitoring & Metrics

### 3.1 Health Check

```
GET /api/health
Expected: {"status": "ok"}
```

Use as the load balancer / uptime monitor probe.

### 3.2 Metrics Endpoint

```
GET /api/[metrics-path]
```

[Describe what the metrics endpoint returns — counts, distributions, etc.]

### 3.3 Log Monitoring

```bash
# Follow service logs
docker compose logs [service] -f

# Filter for errors
docker compose logs [service] --tail 200 | grep -i "error\|failed\|exception"
```

### 3.4 Stuck / Hung Jobs

[Describe how to identify and recover from stuck background jobs]

```bash
# Check for stuck jobs
[command or API call]

# Restart worker to re-queue
docker compose restart [worker-service]
```

---

## 4. Incident Response

### 4.1 API Not Responding

1. `docker compose ps` — check container status
2. `docker compose logs [api-service] --tail 100` — check for errors
3. `ss -tlnp | grep [PORT]` — check port binding
4. `docker compose restart [api-service]`

### 4.2 Worker Not Processing

1. Verify worker is running: `docker compose ps [worker-service]`
2. Check worker logs for errors
3. Verify the worker and API share the same queue path/connection
4. `docker compose restart [worker-service]`

### 4.3 Database Errors

Common causes:
- **"database is locked"** — multiple writers; ensure only one writer process
- **"no such table"** — migrations not applied; run `upgrade head`
- **"disk full"** — `df -h`; free space or move data directory

### 4.4 CORS Errors (browser console)

1. Check `CORS_ORIGINS` in `.env` — ensure the frontend origin is listed
2. `docker compose restart [api-service]`
3. Verify: `curl -H "Origin: [YOUR_ORIGIN]" -I http://[HOST]:[PORT]/api/health`

### 4.5 Rate Limit Errors (429 in production)

1. Review rate limit configuration in the API source
2. Check if the limiter state needs to be reset (in-memory limiters reset on restart)
3. Tune limits for your expected traffic in the config

---

## 5. Capacity & Scaling Guidance

| Signal | Threshold | Action |
|---|---|---|
| [DB size] | > [X] | [Migration action] |
| [Record count] | > [N] | [Add pagination / indexing] |
| [Concurrent users] | > [N] | [Add auth / horizontal scale] |
| [Queue depth] | > [N] tasks | [Add worker threads or instances] |
| [API p95 latency] | > [X] ms | [Profile, optimise, or scale] |

---

## 6. Maintenance Tasks

### 6.1 Dependency Updates

```bash
# Backend
source venv/bin/activate
pip list --outdated
# Update requirements.txt, then test:
python -m pytest tests/ -v

# Frontend
cd web && npm outdated
npm update
npm test
```

### 6.2 Log Rotation

Configure Docker log rotation in `/etc/docker/daemon.json`:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  }
}
```

### 6.3 [Any application-specific maintenance tasks]

[e.g. rebuild full-text search index, clear session store, rotate API keys, etc.]

```bash
[command]
```
