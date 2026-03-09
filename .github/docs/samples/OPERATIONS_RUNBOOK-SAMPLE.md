# Meitheal — Operations Runbook
**Version:** 1.0  
**Last updated:** 2026-03-08

This runbook covers deployment, backup, monitoring, metrics, and incident response procedures for the Meitheal research-paper catalog.

---

## 1. Deployment

### 1.1 Production Deployment (Docker)

```bash
# Pull latest code
git pull origin main

# Build and restart the stack (zero-downtime: depends on load balancer setup)
docker compose down
docker compose up --build -d

# Verify services are up
docker compose ps
docker compose logs api --tail 50
docker compose logs worker --tail 50
```

**Post-deploy smoke check:**
```bash
curl -s http://localhost:8000/api/health | python -m json.tool
# Expected: {"status": "ok"}
```

### 1.2 Environment Variables

All runtime configuration is in `.env`. Before deploying to a new environment:

```bash
cp .env.example .env
# Edit .env — set LLM_API_KEY, CORS_ORIGINS for production domain
```

Critical production overrides:

| Variable | Dev default | Production value |
|---|---|---|
| `DATABASE_URL` | `sqlite+aiosqlite:///./data/catalog.db` | Absolute path: `sqlite+aiosqlite:////app/data/catalog.db` |
| `CORS_ORIGINS` | `["http://localhost:3000","http://localhost:3001","http://localhost:3002"]` | `["https://your-app.example.com"]` |
| `LLM_API_KEY` | _(empty — disables summariser/tagger)_ | Your hosted LLM key |
| `DEBUG` | `false` | `false` (never `true` in production) |

### 1.3 Database Migrations

Run after every deployment that includes schema changes:

```bash
docker compose exec api alembic upgrade head
```

Check current revision:

```bash
docker compose exec api alembic current
```

### 1.4 Rollback

To rollback one migration:
```bash
docker compose exec api alembic downgrade -1
```

To rollback to a specific revision:
```bash
docker compose exec api alembic downgrade <revision_id>
```

---

## 2. Backup & Recovery

### 2.1 Database Backup (SQLite)

The SQLite database lives at `data/catalog.db` inside the Docker volume. Back it up while the API is idle (or use SQLite's safe online backup mode):

```bash
# Option A: Copy while stack is down
docker compose down
cp data/catalog.db data/catalog.db.backup-$(date +%Y%m%d-%H%M%S)
docker compose up -d

# Option B: Online backup using SQLite CLI (safe while API is running)
sqlite3 data/catalog.db ".backup 'data/catalog.db.backup-$(date +%Y%m%d-%H%M%S)'"
```

### 2.2 PDF Storage Backup

Uploaded PDFs are stored in `data/papers/`. Back up this directory with your preferred file backup method:

```bash
tar -czf data-papers-backup-$(date +%Y%m%d).tar.gz data/papers/
```

### 2.3 Restoration

```bash
# Stop the stack
docker compose down

# Restore the database
cp data/catalog.db.backup-YYYYMMDD-HHMMSS data/catalog.db

# Restore PDFs
tar -xzf data-papers-backup-YYYYMMDD.tar.gz

# Restart
docker compose up -d
```

---

## 3. Monitoring & Metrics

### 3.1 Built-in Metrics Endpoint

The API exposes a metrics summary at:

```
GET /api/metrics
```

Response includes total paper count, job status distribution, pipeline stage counts, and top tags. Use this for a quick system health snapshot.

### 3.2 Health Check

```
GET /api/health
```

Returns `{"status": "ok"}` when the API and database connection are healthy. Use this as the load balancer / uptime monitor probe.

### 3.3 Log Monitoring

```bash
# Follow API logs
docker compose logs api -f

# Follow worker logs
docker compose logs worker -f

# Filter for errors
docker compose logs api --tail 200 | grep -i error
docker compose logs worker --tail 200 | grep -i "error\|failed\|exception"
```

### 3.4 Stuck Jobs

Jobs that reach `PROCESSING` status but never complete indicate a worker failure:

```bash
# Check for stuck jobs via the API
curl -s http://localhost:8000/api/jobs | python -m json.tool | grep -A5 '"status": "PROCESSING"'

# Check worker process
docker compose ps worker
docker compose logs worker --tail 50
```

To manually re-queue a stuck job, restart the worker process:
```bash
docker compose restart worker
```

If the worker continues to fail, check `data/huey.db` for queue errors:
```bash
sqlite3 data/huey.db "SELECT * FROM task WHERE error IS NOT NULL ORDER BY created_at DESC LIMIT 10;"
```

---

## 4. Incident Response

### 4.1 API Not Responding

1. Check container status: `docker compose ps`
2. Check API logs: `docker compose logs api --tail 100`
3. Check port binding: `ss -tlnp | grep 8000`
4. Restart API: `docker compose restart api`
5. If still failing, rebuild: `docker compose up --build api`

### 4.2 Worker Not Processing

1. Check worker status: `docker compose ps worker`
2. Check worker logs: `docker compose logs worker --tail 100`
3. Verify `HUEY_DB_PATH` is the same path in both API and worker configs
4. Restart worker: `docker compose restart worker`
5. If tasks are queued but not processing, check for file permission issues on `data/`

### 4.3 Database Errors

Common causes:
- **"database is locked"** — multiple writer processes; ensure only one API instance and one worker are running
- **"no such table"** — migrations not applied; run `alembic upgrade head`
- **"disk full"** — check `df -h`; free space or move `data/` to a larger volume

### 4.4 CORS Errors (browser console)

If browser shows `Access to fetch at '...' from origin '...' has been blocked by CORS policy`:
1. Check `CORS_ORIGINS` in `.env` — ensure your frontend origin is listed
2. Restart the API after changing `.env`: `docker compose restart api`
3. Verify: `curl -H "Origin: https://your-app.example.com" -I http://localhost:8000/api/health`

### 4.5 Rate Limit Errors (429)

If API returns `429 Too Many Requests` for legitimate traffic:
1. Check `src/api/upload.py` and `src/api/arxiv.py` for rate limiter configuration
2. The default limits are suitable for single-user MVP; tune for multi-user scenarios
3. In test environments: ensure `conftest.py` resets **all** `RateLimiter` instances before each test

---

## 5. Capacity & Scaling Guidance

| Signal | Threshold | Action |
|---|---|---|
| SQLite DB size | > 500 MB | Plan Postgres migration |
| Paper count | > 500 | Enable pagination on `/api/papers` |
| Concurrent users | > 1 | Add auth layer; consider Postgres |
| Worker queue depth | > 50 tasks | Add `-w` workers to Huey consumer |
| API response p95 | > 2s | Profile with `py-spy` or FastAPI middleware |

---

## 6. Maintenance Tasks

### 6.1 Dependency Updates

```bash
source venv/bin/activate
pip list --outdated
# Review and update requirements.txt, then test
python -m pytest tests/ -v
```

### 6.2 Log Rotation

Docker log rotation should be configured in `/etc/docker/daemon.json`:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  }
}
```

### 6.3 FTS5 Index Rebuild

If full-text search returns stale results after a bulk import:

```bash
sqlite3 data/catalog.db "INSERT INTO papers_fts(papers_fts) VALUES('rebuild');"
```
