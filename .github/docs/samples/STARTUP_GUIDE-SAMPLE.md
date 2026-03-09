# Meitheal — Developer & Operator Startup Guide

This guide covers everything required to run the Meitheal research-paper catalog locally, either inside Docker (recommended) or natively on your machine.

---

## 1. Prerequisites

| Tool | Minimum version | Purpose |
|---|---|---|
| Python | 3.12+ | Backend API and worker |
| Node.js | 18+ | Next.js frontend (`web/`) |
| npm | 9+ | Frontend package management |
| Docker | 24+ | Containerised stack |
| Docker Compose | v2 (plugin) | Multi-service orchestration |
| git | any | Source control |

Verify before starting:

```bash
python --version        # Python 3.12.x
node --version          # v18.x or later
docker compose version  # Docker Compose version v2.x
```

---

## 2. Quick Start (Docker — recommended)

This is the fastest path. Docker Compose builds the image, runs migrations, and starts both the API server and the background worker in a single command.

### 2.1 Clone the repository

```bash
git clone <repo-url> meitheal
cd meitheal
```

### 2.2 Configure environment variables

```bash
cp .env.example .env
```

Open `.env` and review the values. For a default local run no changes are required, but ensure the following are correct for your environment:

| Variable | What to set |
|---|---|
| `DATABASE_URL` | Leave as-is for container use (`sqlite+aiosqlite:////app/data/catalog.db`) |
| `HUEY_DB_PATH` | Leave as-is (`/app/data/huey.db`) |
| `STORAGE_DIR` | Leave as-is (`/app/data/papers`) |
| `LLM_API_KEY` | Set to your hosted LLM API key; leave blank to disable summarisation and tagging |

### 2.3 Build and start the stack

```bash
docker compose up --build
```

On first boot the `api` service automatically runs `alembic upgrade head` before starting uvicorn.

### 2.4 Access the application

| Service | URL |
|---|---|
| REST API | http://localhost:8000 |
| Interactive API docs (Swagger) | http://localhost:8000/docs |
| Alternative API docs (ReDoc) | http://localhost:8000/redoc |
| Health check | http://localhost:8000/api/health |

### 2.5 Frontend

The Next.js frontend (`web/`) is deployed separately. For production, deploy to Vercel or another Node.js host. For local development alongside the Docker stack see §3.2 below.

---

## 3. Local Development (without Docker)

Use this path when you need fast iteration on backend or frontend code without rebuilding Docker images.

### 3.1 Backend

#### Create and activate a virtual environment

```bash
python -m venv venv
source venv/bin/activate      # macOS/Linux
# venv\Scripts\activate       # Windows
```

Always verify you are inside the venv before running anything:

```bash
which python    # should point to .../venv/bin/python
```

#### Install Python dependencies

```bash
pip install -r requirements.txt
```

#### Configure environment variables

```bash
cp .env.example .env
```

For local development (no Docker), update the path-based variables to use relative paths:

```dotenv
DATABASE_URL=sqlite+aiosqlite:///./data/catalog.db
HUEY_DB_PATH=./data/huey.db
STORAGE_DIR=./data/papers
```

The `data/` directory is created automatically by the FastAPI lifespan hook on first start.

#### Run database migrations

```bash
alembic upgrade head
```

> Note: `alembic.ini` does **not** hard-code a database URL. Alembic reads `DATABASE_URL` from your `.env` file via `alembic/env.py` and the `Settings` class, so the `.env` file must be present before running migrations.

#### Start the API server

```bash
uvicorn src.main:app --reload --port 8000
```

`--reload` watches source files and restarts the server on changes.

#### Start the background worker (separate terminal)

Open a second terminal, activate the venv, then run:

```bash
source venv/bin/activate
python -m huey.bin.huey_consumer src.worker.tasks.huey
```

Add `-w 2` (or higher) to run multiple worker threads for concurrent task processing:

```bash
python -m huey.bin.huey_consumer src.worker.tasks.huey -w 2
```

The worker uses a SQLite-backed Huey queue stored at `HUEY_DB_PATH`.

---

### 3.2 Frontend

```bash
cd web
npm install
```

Copy the frontend environment file:

```bash
cp .env.example .env.local
```

The only variable needed locally is:

```dotenv
NEXT_PUBLIC_API_URL=http://localhost:8000/api
```

Change the host/port if your API is running elsewhere.

Start the development server:

```bash
npm run dev
```

Next.js starts on **http://localhost:3000** by default. If port 3000 is in use, Next.js will automatically try 3001.

> **E2E testing note:** Playwright tests run the frontend on port **3002** by default. This is controlled by the `FRONTEND_PORT` environment variable in `web/playwright.config.ts`. When running Playwright tests, start the dev server with `FRONTEND_PORT=3002 npm run dev` or set `FRONTEND_PORT=3002` in `web/.env.local`.

---

## 4. Environment Variables Reference

All variables are read by `src/config.py` (`pydantic-settings`). Any variable can be set in `.env` or as a shell export.

### Backend (`.env`)

| Variable | Default | Required | Description |
|---|---|---|---|
| `DEBUG` | `false` | No | Enable FastAPI debug mode |
| `DATABASE_URL` | `sqlite+aiosqlite:///./data/catalog.db` | Yes | SQLAlchemy async DB URL. Use 4 slashes for absolute paths (`////app/data/...`) inside Docker |
| `HUEY_DB_PATH` | `./data/huey.db` | Yes | SQLite file used by the Huey task queue |
| `STORAGE_DIR` | `./data/papers` | Yes | Directory where uploaded PDF files are stored |
| `LLM_API_KEY` | _(empty)_ | No | API key for the hosted LLM used by the summariser and tagger. Leave blank to skip those pipeline stages |
| `ARXIV_API_BASE_URL` | `https://export.arxiv.org/api/query` | No | Override the arXiv Atom feed endpoint |
| `APP_TITLE` | `Research Paper Catalog` | No | Title shown in OpenAPI docs |
| `APP_VERSION` | `0.1.0` | No | Version string shown in OpenAPI docs |
| `CORS_ORIGINS` | `["http://localhost:3000","http://localhost:3001","http://localhost:3002"]` | No | Comma-separated list of allowed browser origins. **Must be set to your frontend URL in production.** |

### Frontend (`web/.env.local`)

| Variable | Default | Description |
|---|---|---|
| `NEXT_PUBLIC_API_URL` | `http://localhost:8000/api` | Base URL of the backend REST API, as seen from the browser |

---

## 5. Database Migrations

Meitheal uses [Alembic](https://alembic.sqlalchemy.org/) for schema migrations against the SQLite database.

### Apply all pending migrations

```bash
alembic upgrade head
```

### Check current schema revision

```bash
alembic current
```

### View migration history

```bash
alembic history --verbose
```

### Create a new migration

After adding or modifying SQLAlchemy models in `src/db/models.py`, auto-generate a migration:

```bash
alembic revision --autogenerate -m "describe your change here"
```

Review the generated file in `alembic/versions/` before applying it. Auto-generation is not always perfect — manually verify any `op.drop_*` calls.

Then apply:

```bash
alembic upgrade head
```

### Rollback one revision

```bash
alembic downgrade -1
```

---

## 6. Running Tests

The test suite uses **pytest** with `asyncio_mode = auto` (configured in `pytest.ini`).

```bash
# Activate venv first
source venv/bin/activate

# Run all tests with verbose output
python -m pytest tests/ -v
```

Run a specific test file:

```bash
python -m pytest tests/api/test_metrics.py -v
```

Run tests matching a keyword:

```bash
python -m pytest tests/ -v -k "duplicate"
```

> Tests require the Python dependencies from `requirements.txt` (includes `pytest==8.3.5` and `pytest-asyncio==0.24.0`). No separate test-requirements file is needed.

### Frontend Unit Tests (Vitest)

```bash
cd web
npm test
```

Run in watch mode during development:

```bash
npm run test:watch
```

### E2E Tests (Playwright)

Playwright tests require both the backend and frontend to be running.

1. Start the backend (see §3.1 above): `uvicorn src.main:app --reload --port 8000`
2. Start the Huey worker (see §3.1 above)
3. Start the frontend on port 3002:

```bash
cd web
FRONTEND_PORT=3002 npm run dev
```

4. In a separate terminal, run Playwright:

```bash
cd web
npx playwright test
```

> **Port note:** Playwright is configured to use port 3002 via the `FRONTEND_PORT` environment variable in `playwright.config.ts`. Ensure the frontend dev server is started on the same port.

---

## 7. API Documentation

FastAPI generates interactive documentation automatically. No extra tooling is required.

| Interface | URL | Description |
|---|---|---|
| Swagger UI | http://localhost:8000/docs | Interactive — try endpoints directly in the browser |
| ReDoc | http://localhost:8000/redoc | Read-only reference docs |
| OpenAPI JSON | http://localhost:8000/openapi.json | Machine-readable spec for import into Postman, etc. |

---

## 8. Stopping the Stack

### Docker Compose

```bash
# Graceful stop (keeps volumes and containers)
docker compose down

# Stop and remove all volumes (deletes the SQLite database and PDFs)
docker compose down -v
```

Or press **Ctrl-C** in the terminal running `docker compose up` to stop all services.

### Local development

Press **Ctrl-C** in each terminal:
- Terminal 1: stops `uvicorn`
- Terminal 2: stops the Huey worker

---

## 9. Common Issues

### Port 8000 is already in use

Another process is listening on port 8000.

```bash
# Find and kill the process occupying the port
lsof -ti:8000 | xargs kill -9
```

Or start uvicorn on a different port:

```bash
uvicorn src.main:app --reload --port 8001
```

### Port 3000 is already in use (frontend)

Next.js will automatically try port 3001. Check the terminal output for the actual URL. Update `NEXT_PUBLIC_API_URL` in `web/.env.local` accordingly if the API port also changed.

### Alembic cannot find the database URL

`alembic.ini` intentionally leaves `sqlalchemy.url` blank. Alembic reads the URL from your `.env` file. Ensure:

1. `.env` exists in the project root.
2. `DATABASE_URL` is set and points to a writable path.
3. The venv is active when running `alembic` commands.

### FTS5 not available

The full-text search migration (`b3c4d5e6f7a8_add_fts5_paper_search.py`) requires SQLite compiled with the FTS5 extension. This is included in the official `python:3.12-slim` Docker image. If you see `no such module: fts5` in a local environment, verify your SQLite build:

```bash
python -c "import sqlite3; conn = sqlite3.connect(':memory:'); conn.execute('CREATE VIRTUAL TABLE t USING fts5(x)'); print('FTS5 OK')"
```

On macOS, the system Python ships with an older SQLite. Install Python via [pyenv](https://github.com/pyenv/pyenv) or [Homebrew](https://brew.sh/) to get a version linked against a modern SQLite with FTS5.

### Worker not processing tasks

Verify the worker process is running and reading from the same `HUEY_DB_PATH` as the API. A common mistake in local development is having different paths in the API process and the worker process due to relative vs absolute path handling.

```bash
# Both should print the same resolved path
python -c "from src.config import get_settings; print(get_settings().huey_db_path)"
```

### ImportError / ModuleNotFoundError

The venv is not active, or dependencies are out of date.

```bash
source venv/bin/activate
pip install -r requirements.txt
```
