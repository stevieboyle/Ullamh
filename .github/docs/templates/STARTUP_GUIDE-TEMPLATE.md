# [PROJECT_NAME] — Developer & Operator Startup Guide

> **Usage:** Fill in this template when first setting up a project. Keep it updated as ports, commands, or tooling change. This is the first document new developers and operators should read. After completing the guide, verify every code block works on a clean checkout.

---

## 1. Prerequisites

| Tool | Minimum version | Purpose |
|---|---|---|
| Python | [3.x+] | Backend API and worker |
| Node.js | [18+] | Frontend |
| npm | [9+] | Frontend package management |
| Docker | [24+] | Containerised stack |
| Docker Compose | v2 (plugin) | Multi-service orchestration |
| git | any | Source control |

Verify before starting:

```bash
python --version        # Python [3.x.x]
node --version          # v[18.x] or later
docker compose version  # Docker Compose version v2.x
```

---

## 2. Quick Start (Docker — recommended)

### 2.1 Clone the repository

```bash
git clone [repo-url] [project-dir]
cd [project-dir]
```

### 2.2 Configure environment variables

```bash
cp .env.example .env
```

Open `.env` and review the values. For a default local run the following are required:

| Variable | What to set |
|---|---|
| `DATABASE_URL` | [description and example value] |
| `[REQUIRED_VAR]` | [description] |
| `[OPTIONAL_VAR]` | [Set to your key; leave blank to disable optional feature] |

### 2.3 Build and start the stack

```bash
docker compose up --build
```

On first boot, [describe any automatic setup steps, e.g. migrations that run].

### 2.4 Access the application

| Service | URL |
|---|---|
| REST API | http://localhost:[PORT] |
| API docs (Swagger) | http://localhost:[PORT]/docs |
| Health check | http://localhost:[PORT]/api/health |
| Frontend | http://localhost:[FRONTEND_PORT] |

### 2.5 Frontend (if deployed separately)

[Describe how the frontend is deployed — Vercel, separate container, or alongside the API.]

---

## 3. Local Development (without Docker)

Use this path when you need fast iteration without rebuilding Docker images.

### 3.1 Backend

#### Create and activate a virtual environment

```bash
python -m venv venv
source venv/bin/activate      # macOS/Linux
# venv\Scripts\activate       # Windows
```

Verify you are inside the venv:

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

For local development (no Docker), update path-based variables to use relative paths:

```dotenv
[VAR_1]=[relative-path-example]
[VAR_2]=[relative-path-example]
```

#### Run database migrations

```bash
[migration-tool] upgrade head
```

#### Start the API server

```bash
[api-server-command] --reload --port [PORT]
```

#### Start the background worker (separate terminal)

```bash
source venv/bin/activate
[worker-start-command]
```

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

The key variable for local dev:

```dotenv
[NEXT_PUBLIC_API_URL or equivalent]=http://localhost:[BACKEND_PORT]/api
```

Start the development server:

```bash
npm run dev
```

Frontend starts on **http://localhost:[FRONTEND_PORT]** by default.

> **E2E testing note:** [Describe any different port required for E2E tests, e.g. FRONTEND_PORT=3002 for Playwright]

---

## 4. Environment Variables Reference

### Backend (`.env`)

| Variable | Default | Required | Description |
|---|---|---|---|
| `[VAR_1]` | `[default]` | Yes | [Description] |
| `[VAR_2]` | `[default]` | Yes | [Description] |
| `[VAR_3]` | _(empty)_ | No | [Optional feature — disabled if blank] |
| `CORS_ORIGINS` | `["http://localhost:[PORT]"]` | No | Comma-separated list of allowed browser origins. **Must be set for staging/production.** |

### Frontend (`web/.env.local`)

| Variable | Default | Description |
|---|---|---|
| `[API_URL_VAR]` | `http://localhost:[PORT]/api` | Base URL of the backend REST API |

---

## 5. Database Migrations

### Apply all pending migrations

```bash
[migration-tool] upgrade head
```

### Check current schema revision

```bash
[migration-tool] current
```

### Create a new migration

```bash
[migration-tool] revision --autogenerate -m "describe your change here"
```

Review the generated file before applying. Manually verify any `drop_*` operations.

```bash
[migration-tool] upgrade head
```

### Rollback one revision

```bash
[migration-tool] downgrade -1
```

---

## 6. Running Tests

### Backend

```bash
source venv/bin/activate
python -m pytest tests/ -v

# Run a specific file
python -m pytest tests/[module]/test_[feature].py -v

# Run matching a keyword
python -m pytest tests/ -v -k "[keyword]"
```

### Frontend Unit Tests

```bash
cd web
npm test
```

### E2E Tests (Playwright)

Requires both backend and frontend running.

1. Start the backend: `[api-server-command] --reload --port [PORT]`
2. Start the background worker: `[worker-start-command]`
3. Start the frontend on the E2E port:

```bash
cd web
[FRONTEND_PORT_VAR]=[E2E_PORT] npm run dev
```

4. Run Playwright:

```bash
cd web
npx playwright test
```

---

## 7. API Documentation

| Interface | URL | Description |
|---|---|---|
| Swagger UI | http://localhost:[PORT]/docs | Interactive — try endpoints in the browser |
| ReDoc | http://localhost:[PORT]/redoc | Read-only reference docs |
| OpenAPI JSON | http://localhost:[PORT]/openapi.json | Machine-readable spec |

---

## 8. Stopping the Stack

### Docker Compose

```bash
docker compose down           # graceful stop (keeps data)
docker compose down -v        # stop and delete all volumes (deletes database)
```

### Local development

Press **Ctrl-C** in each terminal:
- Terminal 1: stops the API server
- Terminal 2: stops the background worker

---

## 9. Common Issues

### Port [PORT] is already in use

```bash
lsof -ti:[PORT] | xargs kill -9
```

Or start on a different port:

```bash
[api-server-command] --reload --port [ALTERNATE_PORT]
```

### Migrations cannot find the database URL

1. Ensure `.env` exists in the project root.
2. Verify `[DATABASE_URL_VAR]` is set and the path is writable.
3. Ensure the venv is active.

### Worker not processing tasks

Verify the worker and API are using the same queue path/connection. A common mistake in local development is path mismatches due to relative vs absolute paths.

### ImportError / ModuleNotFoundError

The venv is not active or dependencies are out of date.

```bash
source venv/bin/activate
pip install -r requirements.txt
```
