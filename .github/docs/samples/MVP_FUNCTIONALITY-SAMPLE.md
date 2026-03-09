# MVP Functionality Description — Research Paper Catalog
**Epic:** KAN-4 — MVP: Research paper ingestion, cataloging, and citation lineage  
**Author:** @Analyst  
**Date:** 2026-03-08  
**Status:** Complete — all stories Done

---

## Table of Contents

- [MVP Functionality Description — Research Paper Catalog](#mvp-functionality-description--research-paper-catalog)
  - [Table of Contents](#table-of-contents)
  - [1. Overview](#1-overview)
  - [2. Paper Ingestion](#2-paper-ingestion)
    - [2.1 PDF Upload (KAN-10)](#21-pdf-upload-kan-10)
    - [2.2 arXiv URL Ingestion (KAN-12)](#22-arxiv-url-ingestion-kan-12)
    - [2.3 Duplicate Detection (KAN-23)](#23-duplicate-detection-kan-23)
  - [3. Async Processing Pipeline](#3-async-processing-pipeline)
    - [3.1 Task Queue Infrastructure (KAN-9)](#31-task-queue-infrastructure-kan-9)
    - [3.2 PDF Text Extraction (KAN-14)](#32-pdf-text-extraction-kan-14)
    - [3.3 Text Chunking (KAN-15)](#33-text-chunking-kan-15)
    - [3.4 AI Summarization (KAN-16)](#34-ai-summarization-kan-16)
    - [3.5 Tag Generation (KAN-17)](#35-tag-generation-kan-17)
    - [3.6 Reference Extraction (KAN-18)](#36-reference-extraction-kan-18)
    - [3.7 Reference Resolution \& Citation Graph (KAN-19)](#37-reference-resolution--citation-graph-kan-19)
  - [4. Job Status Tracking (KAN-13)](#4-job-status-tracking-kan-13)
  - [5. Library \& Search (KAN-20)](#5-library--search-kan-20)
  - [6. Paper Detail View (KAN-21)](#6-paper-detail-view-kan-21)
  - [7. Citation Lineage View (KAN-22)](#7-citation-lineage-view-kan-22)
  - [8. Observability \& Metrics (KAN-24)](#8-observability--metrics-kan-24)
  - [9. Infrastructure \& Platform](#9-infrastructure--platform)
    - [9.1 Backend Scaffold (KAN-5)](#91-backend-scaffold-kan-5)
    - [9.2 Frontend Scaffold (KAN-6)](#92-frontend-scaffold-kan-6)
    - [9.3 Database \& Migrations (KAN-7)](#93-database--migrations-kan-7)
    - [9.4 File Storage (KAN-8)](#94-file-storage-kan-8)
    - [9.5 Core API Endpoints (KAN-11)](#95-core-api-endpoints-kan-11)
    - [9.6 Containerisation (KAN-36)](#96-containerisation-kan-36)
    - [9.7 Technical Decisions Spike (KAN-37)](#97-technical-decisions-spike-kan-37)
  - [10. Test Coverage (KAN-38)](#10-test-coverage-kan-38)
    - [Backend (pytest) — 86 tests](#backend-pytest--86-tests)
    - [Frontend (Vitest) — 19 tests](#frontend-vitest--19-tests)
    - [E2E (Playwright) — 13 tests](#e2e-playwright--13-tests)

---

## 1. Overview

The KAN-4 MVP is a three-tier web application for ingesting, processing, and cataloging academic research papers. Users can add papers either by uploading a PDF directly or by providing an arXiv URL. Each paper is processed asynchronously through a multi-stage pipeline that extracts text, generates summaries and tags, and builds a citation graph by resolving bibliography references to canonical external works.

The system is composed of:

- A **Next.js 16** frontend (TypeScript, App Router, TailwindCSS)
- A **FastAPI** backend (Python, Pydantic v2, SQLAlchemy)
- A **Huey** background worker for async pipeline execution
- A **SQLite** database (with FTS5 for full-text search, Alembic migrations)
- A **local filesystem** PDF store (behind a storage abstraction layer)

All three backend processes share a single `./data/` directory on the host for the database, task queue, and PDF files.

---

## 2. Paper Ingestion

### 2.1 PDF Upload (KAN-10)

**`POST /api/papers/upload`** — accepts a `multipart/form-data` request with a PDF file.

On receipt the API:

1. Validates the file is a PDF (MIME type check).
2. Computes a SHA-256 checksum of the file bytes.
3. Checks `paper_files.checksum_sha256` for a pre-existing match — if found, returns the existing `paper_id` immediately (see §2.3).
4. Saves the PDF to `./data/papers/{paper_id}.pdf` via the `LocalStorageBackend`.
5. Inserts a `Paper` row with `source=UPLOAD`.
6. Inserts a `PaperFile` row with `storage_path`, `checksum_sha256`, and `size_bytes`.
7. Inserts a `Job` row with `status=QUEUED`.
8. Enqueues a `process_paper` Huey task.
9. Returns `{paper_id, job_id}` — the frontend begins polling for job status.

A rate limit of 10 requests/minute per IP is enforced via `slowapi`.

The Add Paper page (`/add`) presents a tabbed interface where the PDF tab drives this flow. After a successful submission, the `JobStatusPoller` component begins live polling.

### 2.2 arXiv URL Ingestion (KAN-12)

**`POST /api/papers/arxiv`** — accepts `{"arxiv_url": "..."}` with either a bare arXiv ID (`1706.03762`) or a full `https://arxiv.org/abs/...` URL.

The API:

1. Normalises the input to extract the canonical arXiv ID (strips version suffix, e.g. `v3`).
2. Checks `papers.arxiv_id` — if already in the catalog, returns the existing `paper_id` immediately.
3. Calls the arXiv Atom API (`export.arxiv.org/api/query?id_list={id}`) to retrieve:
   - Title, abstract, `published` date, author list, category terms.
4. Downloads the PDF from `arxiv.org/pdf/{id}`.
5. Computes SHA-256 checksum and checks for a content-level duplicate (see §2.3).
6. On a new paper:
   - Inserts `Paper` with `source=ARXIV`, `arxiv_id`, title, abstract, and `published_year`.
   - Inserts `PaperAuthor` rows (one per `<author>` in the Atom feed, preserving order).
   - Inserts `Tag` + `PaperTag` rows for each `<category>` term (tagged `source=ARXIV`).
   - Writes the PDF to the filesystem.
   - Inserts `PaperFile`, `Job` (QUEUED), and enqueues `process_paper`.
7. Returns `{paper_id, job_id}`.

The same 10 req/min rate limit applies. The arXiv tab on `/add` drives this flow.

### 2.3 Duplicate Detection (KAN-23)

Duplicates are caught at two independent levels:

| Level | Field checked | Scope |
|---|---|---|
| arXiv ID | `papers.arxiv_id` | arXiv ingestion only |
| PDF content | `paper_files.checksum_sha256` | Both upload and arXiv paths |

If either check finds an existing record, the API returns the existing `paper_id` with HTTP 200 (not 201) and does **not** enqueue a new worker task. This guarantees idempotency: re-submitting the same paper never creates duplicate records or redundant processing jobs.

---

## 3. Async Processing Pipeline

### 3.1 Task Queue Infrastructure (KAN-9)

All heavy processing runs outside the HTTP request cycle in a **Huey** background worker backed by a SQLite task queue (`./data/huey.db`). The worker is a separate process (`huey_consumer`) that dequeues and executes `process_paper(paper_id, job_id)` tasks.

The pipeline is designed to be **idempotent**: if a paper is already in `PROCESSED` status the task exits immediately without re-running stages.

Job status transitions: `QUEUED → RUNNING → PROCESSED | FAILED`

Stage-level progress is also tracked on the `jobs.stage` column:  
`FETCH → PARSE → CHUNK → SUMMARIZE → TAG → REF_EXTRACT → REF_RESOLVE → INDEX`

If any stage raises an unhandled exception the job is moved to `FAILED` with the stage name and error message recorded, allowing precise diagnosis.

### 3.2 PDF Text Extraction (KAN-14)

**Stage: PARSE**

The worker reads the PDF from `paper_files.storage_path` and extracts full text using **PyMuPDF (fitz)**. The extracted text is stored in the `paper_text` table (`full_text` column). For arXiv papers the metadata (title, abstract, authors, year) is already populated at ingest time; for PDF uploads this stage also populates those fields via best-effort heuristic extraction.

The `PdfExtractor` service (`src/services/pdf_extractor.py`) encapsulates the PyMuPDF call, providing a clean contract for the worker task.

### 3.3 Text Chunking (KAN-15)

**Stage: CHUNK**

The full text is split into overlapping fixed-size chunks by the `Chunker` service (`src/services/chunker.py`). Each chunk is stored as a `paper_chunks` row with `chunk_index`, `text`, `start_offset`, and `end_offset`.

Chunk parameters (size and overlap) are configurable. A guard prevents `overlap >= chunk_size` from causing an infinite loop — this was a production bug identified and fixed during QA.

Chunks serve as the searchable units for the FTS5 index and are the retrieval primitive for any future RAG integration.

### 3.4 AI Summarization (KAN-16)

**Stage: SUMMARIZE**

The `Summarizer` service (`src/services/summarizer.py`) calls an LLM API to produce two outputs stored in `paper_summaries`:

| Field | Content |
|---|---|
| `tldr` | A single-sentence "too long; didn't read" summary |
| `structured_json` | A structured breakdown: Problem / Approach / Results / Limitations |

The LLM model used is recorded in `paper_summaries.model` for traceability. The paper detail page surfaces `tldr` directly; `structured_json` is available for richer rendering.

### 3.5 Tag Generation (KAN-17)

**Stage: TAG**

Tags are gathered from three sources and stored in `paper_tags` with a `source` discriminator:

| Source | Mechanism | `source` value |
|---|---|---|
| arXiv categories | Set at ingest time from Atom `<category>` elements | `ARXIV` |
| Keyword extraction | Extracted from title/abstract during pipeline | `KEYWORD` |
| LLM-generated | Model-generated descriptive tags | `LLM` |

All tags are normalised through a shared `tags` table (unique by text) so the same tag string is shared across papers rather than duplicated.

### 3.6 Reference Extraction (KAN-18)

**Stage: REF_EXTRACT**

The `RefExtractor` service (`src/services/ref_extractor.py`) identifies the bibliography section of the extracted text and parses individual reference entries. Each entry is stored in `paper_references` as:

- `raw_text` — the verbatim bibliography string
- `parsed_json` — structured fields (title, authors, year, DOI, arXiv ID) extracted from the raw text where available
- `ref_index` — position in the bibliography

Unrecognised or malformed entries are stored as `raw_text` only; they remain available for human review.

### 3.7 Reference Resolution & Citation Graph (KAN-19)

**Stage: REF_RESOLVE**

The `RefResolver` service (`src/services/ref_resolver.py`) implements a **precision-first** resolution strategy. For each `paper_references` row it attempts to match the reference to a canonical external work via a waterfall:

1. **DOI** — direct resolution if a DOI is present in `parsed_json`
2. **arXiv ID** — direct lookup if an arXiv ID is present
3. **Title/author/year search** — queries OpenAlex (`api.openalex.org`) first, then CrossRef (`api.crossref.org`) as fallback

A confidence score is computed from string similarity, year match, and author overlap. Only matches above a configured threshold are accepted. Low-confidence matches are stored as `status=UNRESOLVED` rather than silently accepted as false positives.

Resolved references produce:

- A `works` row (canonical work, deduplicated by DOI/arXiv ID across all papers in the catalog)
- A `reference_resolutions` row linking the raw reference to the canonical work
- A `citations` row recording the directed edge `(source_paper_id → target_work_id)`

When a target work is itself in the catalog as a `Paper`, `citations.target_paper_id` is populated, enabling in-library graph traversal.

---

## 4. Job Status Tracking (KAN-13)

**`GET /api/jobs/{job_id}`** and **`GET /api/jobs`**

Every ingestion operation creates a `Job` record. The frontend `JobStatusPoller` component polls `GET /api/jobs/{job_id}` every 2 seconds and renders a live status badge. Once the job reaches a terminal state (`PROCESSED` or `FAILED`) polling stops and the user is presented with a link to the paper detail page (on success) or an error message with the failure stage (on failure).

The `/jobs` page provides a full history of all ingestion jobs, including their final status, stage, and error message. The page auto-refreshes (5-second interval) while any jobs are in a non-terminal state.

The `JobResponse` schema exposes: `id`, `paper_id`, `status`, `stage`, `error`, `created_at`, `updated_at`.

---

## 5. Library & Search (KAN-20)

**`GET /api/papers`**

The library endpoint returns a paginated list of papers with full-text search and structured filters:

| Parameter | Mechanism | Example |
|---|---|---|
| `q` | FTS5 `MATCH` with BM25 ranking | `?q=transformer+attention` |
| `tag` | `paper_tags.tag ILIKE %?%` join | `?tag=machine-learning` |
| `author` | `paper_authors.name ILIKE %?%` join | `?author=LeCun` |
| `year` | `papers.published_year = ?` | `?year=2024` |
| `page` / `page_size` | SQL LIMIT/OFFSET | `?page=2&page_size=20` |

The FTS5 search uses the `paper_search` virtual table, which indexes both `papers.title` and `paper_text.full_text` using a Porter stemmer. Results are ranked by BM25 relevance. An invalid FTS5 query (e.g. unbalanced quotes) gracefully falls back to returning all papers in reverse-chronological order rather than returning a 500 error.

The FTS5 table is kept in sync via SQLite `AFTER INSERT` / `AFTER UPDATE` / `AFTER DELETE` triggers on the `papers` and `paper_text` tables.

The response is a `PaperListResponse` containing `total` (count across all pages) and `items` (array of `PaperListItem` with id, title, source, published_year, authors, and tags).

---

## 6. Paper Detail View (KAN-21)

**`GET /api/papers/{paper_id}`**

Returns a `PaperDetail` response assembling data from multiple tables:

| Field | Source |
|---|---|
| `id`, `title`, `abstract`, `source`, `arxiv_id`, `published_year` | `papers` |
| `authors` | `paper_authors` (ordered) |
| `tags` | `paper_tags` → `tags` |
| `tldr` | `paper_summaries.tldr` |
| `structured_summary` | `paper_summaries.structured_json` |
| `full_text` | `paper_text.full_text` |
| `status` | Latest `jobs.status` for this paper |

Returns HTTP 404 if the paper ID is unknown.

The `/papers/[id]` Next.js page is a Server Component that fetches paper detail, references, and citations in parallel (`Promise.allSettled`) — a failure in the references or citations fetch does not prevent the page from rendering the paper metadata and summary.

**`GET /api/files/{paper_id}`** streams the raw PDF bytes back to the browser, enabling an in-page download link.

---

## 7. Citation Lineage View (KAN-22)

**`GET /api/papers/{paper_id}/references`** — outbound references extracted from the paper's bibliography. Each item includes the `raw_text`, `parsed_json`, and — if resolved — the canonical `Work` record (DOI, title, year, venue, arXiv ID).

**`GET /api/papers/{paper_id}/citations`** — inbound citations: other papers in the catalog that have cited this paper. Each item links back to the source paper.

Both endpoints return HTTP 404 for unknown paper IDs and an empty array when no references/citations exist.

The paper detail page renders these as two distinct sections:
- **References** — all outbound bibliography entries, with resolved works distinguished from unresolved raw text entries
- **Cited By** — inbound citations from within the catalog

---

## 8. Observability & Metrics (KAN-24)

**`GET /api/metrics`**

Returns aggregate counters across the catalog:

| Metric | Description |
|---|---|
| `total_papers` | Total papers in the catalog |
| `processed_papers` | Papers with at least one `PROCESSED` job |
| `failed_papers` | Papers whose latest job is `FAILED` |
| `total_references` | Total `paper_references` rows |
| `resolved_references` | References with `status=RESOLVED` |
| `unresolved_references` | References with `status=UNRESOLVED` |
| `resolution_rate` | `resolved / total` as a float |

These metrics provide at-a-glance health of the ingestion pipeline and the quality of reference resolution.

**`GET /health`** returns `{status: "ok", version, db_url}` for infrastructure health checks.

---

## 9. Infrastructure & Platform

### 9.1 Backend Scaffold (KAN-5)

FastAPI application bootstrapped in `src/main.py` with:

- Modular API routers mounted under `/api` (`upload`, `arxiv`, `papers`, `jobs`, `files`, `metrics`, `health`)
- Pydantic v2 settings via `src/config.py` (reads from `.env`)
- Async SQLAlchemy engine with `aiosqlite` driver
- `slowapi` rate limiting middleware (10 req/min on ingestion endpoints)
- `CORSMiddleware` allowing configurable origins (defaults: `localhost:3000/3001/3002`)
- Lifespan context manager for startup/teardown

### 9.2 Frontend Scaffold (KAN-6)

Next.js 16 App Router application in `web/` with:

- TailwindCSS for styling
- `src/lib/api.ts` — typed fetch helper (`apiFetch`, `uploadPaper`, `ingestArxiv`, `getJobStatus`, `listJobs`, `getPaperDetail`, `getPaperReferences`, `getPaperCitations`)
- Four routes: `/` (library), `/add` (ingest), `/jobs` (history), `/papers/[id]` (detail)
- `JobStatusPoller` client component for live polling

### 9.3 Database & Migrations (KAN-7)

SQLite database at `./data/catalog.db` managed by **Alembic**. Two migrations are in place:

| Migration | Content |
|---|---|
| `8ae388613120` — Initial schema | All core tables: `papers`, `paper_authors`, `paper_files`, `jobs`, `paper_text`, `paper_chunks`, `paper_summaries`, `tags`, `paper_tags`, `paper_references`, `works`, `reference_resolutions`, `citations` |
| `b3c4d5e6f7a8` — FTS5 search | `paper_search` virtual table + 5 auto-sync triggers |

All models use UUID primary keys for `papers` and `jobs` tables, with integer PKs for supporting tables. The SQLAlchemy ORM layer is designed to allow a future mechanical migration to Postgres.

### 9.4 File Storage (KAN-8)

PDFs are stored under `./data/papers/` via a `StorageBackend` abstraction (`src/storage/base.py`). The concrete implementation is `LocalStorageBackend` (`src/storage/local.py`). The abstraction is designed so a future swap to S3-compatible object storage requires only a new backend class, not changes to calling code.

### 9.5 Core API Endpoints (KAN-11)

Full REST surface:

| Method | Path | Description |
|---|---|---|
| `GET` | `/health` | Health check |
| `POST` | `/api/papers/upload` | PDF ingestion |
| `POST` | `/api/papers/arxiv` | arXiv ingestion |
| `GET` | `/api/papers` | Paginated library list + search |
| `GET` | `/api/papers/{id}` | Paper detail |
| `GET` | `/api/papers/{id}/references` | Outbound references |
| `GET` | `/api/papers/{id}/citations` | Inbound citations |
| `GET` | `/api/jobs` | Job history list |
| `GET` | `/api/jobs/{id}` | Single job status |
| `GET` | `/api/files/{id}` | Stream PDF bytes |
| `GET` | `/api/metrics` | Aggregate catalog metrics |

### 9.6 Containerisation (KAN-36)

A `docker-compose.yml` defines two services sharing a `./data` volume:

| Service | Command | Port |
|---|---|---|
| `api` | `alembic upgrade head && uvicorn src.main:app` | 8000 |
| `worker` | `huey_consumer src.worker.tasks.huey` | — |

The `Dockerfile` installs Python dependencies from `requirements.txt` and is based on a slim Python 3.12 image. The frontend is run separately with `npm run dev` for local development.

### 9.7 Technical Decisions Spike (KAN-37)

Closed the open technical decisions from Architecture Spec §15:

| Decision | Resolution |
|---|---|
| PDF extraction library | **PyMuPDF (fitz)** |
| Reference resolver primary | **OpenAlex** (free, no API key, high coverage) |
| Reference resolver fallback | **CrossRef** |
| Auth strategy (MVP) | No auth — single-user local deployment |
| Hosting strategy | Local docker-compose; cloud deployment path documented |

---

## 10. Test Coverage (KAN-38)

A comprehensive automated test suite was built across all layers. All 118 tests pass.

### Backend (pytest) — 86 tests

| File | Tests | Scope |
|---|---|---|
| `tests/api/test_papers.py` | 16 | Papers list, FTS fallback, filters, pagination, detail, 404, summary/text enrichment, job status, references |
| `tests/api/test_arxiv.py` | 9 | arXiv ingest happy path (all DB rows), worker dispatch, URL formats, error paths |
| `tests/api/test_duplicate_detection.py` | 7 | Upload + arXiv idempotency, worker enqueued exactly once |
| `tests/api/test_upload.py` | 7 | PDF upload happy path, non-PDF rejection |
| `tests/api/test_jobs.py` | 7 | Job GET by ID, job list, pagination |
| `tests/api/test_files.py` | 3 | PDF streaming, 404 unknown, 404 missing on disk |
| `tests/api/test_metrics.py` | 2 | Metrics endpoint shape |
| `tests/worker/test_tasks.py` | 12 | Worker happy path, PARSE/CHUNK/SUMMARIZE failure stages, idempotency guard |
| `tests/unit-tests/backend/` | 23 | Unit tests for chunker, pdf_extractor, ref_extractor, ref_resolver, storage_local, summarizer, tagger |

### Frontend (Vitest) — 19 tests

| File | Tests | Scope |
|---|---|---|
| `tests/unit-tests/ui/AddPaperPage.test.tsx` | 4 | PDF submit, arXiv submit, validation, error state |
| `tests/unit-tests/ui/JobStatusPoller.test.tsx` | 6 | Loading, processed, failed, polling lifecycle, unmount |
| `tests/unit-tests/ui/api.test.ts` | 4 | `apiFetch`, `uploadPaper`, `ingestArxiv`, error handling |
| `tests/component/AddPaperPage.component.test.tsx` | 2 | Component boundary tests through real poller/api layer |
| `tests/component/JobsPage.component.test.tsx` | 2 | Active polling, empty state |
| `tests/component/PaperDetailPage.component.test.tsx` | 1 | Partial reference/citation failure resilience |

### E2E (Playwright) — 13 tests

| File | Tests | Scope |
|---|---|---|
| `web/tests/smoke.spec.ts` | 4 | Page load, navigation |
| `web/tests/upload-journey.spec.ts` | 4 | Full upload journey (mocked API) |
| `web/tests/live-stack.spec.ts` | 5 | CORS preflight, health check, real PDF upload, job record, non-PDF rejection (no mocking) |
