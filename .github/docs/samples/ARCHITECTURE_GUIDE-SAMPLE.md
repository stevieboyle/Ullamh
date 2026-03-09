# Architecture Guide — Research Paper Catalog (KAN-4 MVP)

**Author:** @Architect  
**Version:** 1.1  
**Date:** 2026-03-08  

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Deployment Architecture](#2-deployment-architecture)
3. [API Surface Reference](#3-api-surface-reference)
4. [Data Flow: PDF Upload](#4-data-flow-pdf-upload)
5. [Data Flow: arXiv Ingestion](#5-data-flow-arxiv-ingestion)
6. [Deduplication Logic](#6-deduplication-logic)
7. [Worker Pipeline](#7-worker-pipeline)
8. [Reference Resolution Chain](#8-reference-resolution-chain)
9. [Full-Text Search Architecture](#9-full-text-search-architecture)
10. [Database Schema (ER Diagram)](#10-database-schema-er-diagram)
11. [Frontend Architecture](#11-frontend-architecture)
12. [Configuration & Environment](#12-configuration--environment)
13. [Inter-Component Data Contracts](#13-inter-component-data-contracts)

---

## 1. System Overview

The system is a three-tier web application: a **Next.js frontend**, a **FastAPI backend**, and a **Huey background worker**. All three tiers share a single `./data/` directory on the host, giving them access to the SQLite database, the Huey task queue, and the PDF file store.

```mermaid
graph TB
    subgraph Browser["Browser"]
        UI["Next.js 16 (Turbopack)\nport 3002"]
    end

    subgraph Backend["Backend (Python / FastAPI)"]
        API["FastAPI\nport 8000"]
        Worker["Huey Worker\n(sqlite-backed)"]
    end

    subgraph Data["Shared Data Layer (./data/)"]
        DB[("catalog.db\n(SQLite + FTS5)")]
        Queue[("huey.db\n(Huey task queue)")]
        FS["papers/\n(PDF file store)"]
    end

    subgraph ExternalAPIs["External APIs"]
        arXiv["arXiv Atom API\nexport.arxiv.org"]
        OpenAlex["OpenAlex\napi.openalex.org"]
        CrossRef["CrossRef\napi.crossref.org"]
    end

    UI -- "REST / JSON\nhttp://localhost:8000/api" --> API
    API -- "async SQLAlchemy\n(aiosqlite)" --> DB
    API -- "enqueue task" --> Queue
    API -- "read / write PDFs" --> FS
    Worker -- "dequeue task" --> Queue
    Worker -- "async SQLAlchemy" --> DB
    Worker -- "read PDFs\n(PyMuPDF)" --> FS
    API -- "HTTP fetch metadata" --> arXiv
    Worker -- "HTTP resolve references" --> OpenAlex
    Worker -- "HTTP fallback" --> CrossRef
```

---

## 2. Deployment Architecture

### Local Development

```mermaid
graph LR
    subgraph Host["Linux Host"]
        direction TB
        VENV["Python venv (venv/)"]
        NPM["Node.js (npm)"]
        subgraph Processes["Running Processes"]
            P1["uvicorn src.main:app\n--port 8000\nPID ~1500293"]
            P2["huey_consumer src.worker.tasks.huey\n-w 1\nPID ~1500454"]
            P3["next dev\n--port 3002\nPID ~1500585"]
        end
        subgraph Logs["Log Files (/tmp/)"]
            L1["/tmp/api.log"]
            L2["/tmp/worker.log"]
            L3["/tmp/frontend.log"]
        end
        subgraph DataDir["./data/"]
            D1["catalog.db"]
            D2["huey.db (meitheal)"]
            D3["papers/*.pdf"]
        end
    end

    P1 --> L1
    P2 --> L2
    P3 --> L3
    P1 --> D1
    P2 --> D1
    P2 --> D2
    P1 --> D2
    P1 --> D3
    P2 --> D3
```

### Docker / Production

```mermaid
graph TB
    subgraph DockerCompose["docker-compose.yml"]
        direction LR
        subgraph api_svc["Service: api"]
            APICmd["alembic upgrade head\n&& uvicorn ... --port 8000"]
        end
        subgraph worker_svc["Service: worker"]
            WorkerCmd["huey_consumer\nsrc.worker.tasks.huey"]
        end
    end

    subgraph SharedVolume["Volume: ./data → /app/data"]
        VDB[("catalog.db")]
        VQ[("huey.db")]
        VFS["papers/"]
    end

    api_svc -- "depends_on -->" worker_svc
    APICmd --> VDB
    APICmd --> VFS
    WorkerCmd --> VDB
    WorkerCmd --> VQ
    WorkerCmd --> VFS

    Client["HTTP Client\nport 8000"] --> APICmd
```

---

## 3. API Surface Reference

```mermaid
graph LR
    subgraph FastAPI["FastAPI  (src/main.py)"]
        direction TB
        R_Health["GET /health\n→ {status, version, db_url}"]
        R_Upload["POST /api/papers/upload\n multipart/form-data\n→ {paper_id, job_id}"]
        R_Arxiv["POST /api/papers/arxiv\n{arxiv_url}\n→ {paper_id, job_id}"]
        R_Papers["GET /api/papers\n?q, tag, author, year, page\n→ PaperListResponse"]
        R_Paper["GET /api/papers/{id}\n→ PaperDetail"]
        R_Refs["GET /api/papers/{id}/references\n→ ReferenceSchema[]"]
        R_Cites["GET /api/papers/{id}/citations\n→ CitationSchema[]"]
        R_Jobs["GET /api/jobs\n→ JobListResponse"]
        R_Job["GET /api/jobs/{id}\n→ JobResponse"]
        R_File["GET /api/files/{id}\n→ StreamingResponse (PDF)"]
        R_Metrics["GET /api/metrics\n→ MetricsResponse"]
    end

    subgraph Middleware["Middleware (applied at startup)"]
        CORS["CORSMiddleware\nallow_origins=settings.cors_origins\nallow_methods=[*]\nallow_headers=[*]"]
        RL["slowapi Rate Limiter\n10 req/min\n(upload + arxiv)"]
    end

    R_Upload --> RL
    R_Arxiv --> RL
```

---

## 4. Data Flow: PDF Upload

This sequence covers what happens from the moment a user selects a PDF in the browser to the paper appearing in the catalog.

```mermaid
sequenceDiagram
    actor User
    participant UI as "Next.js /add"
    participant API as "FastAPI /api/papers/upload"
    participant DB as "SQLite catalog.db"
    participant FS as "PDF File Store"
    participant Queue as "Huey Queue (huey.db)"
    participant Worker as "Huey Worker"

    User->>UI: Select PDF file, click Upload
    UI->>API: POST /api/papers/upload (multipart, file bytes)

    API->>API: SHA-256 checksum of PDF bytes

    API->>DB: SELECT PaperFile WHERE checksum = ?
    alt Duplicate found
        DB-->>API: Existing PaperFile row
        API-->>UI: {paper_id (existing), job_id: null} 200
        UI-->>User: "Already in catalog" badge
    else New paper
        DB-->>API: No rows

        API->>DB: INSERT Paper (source=UPLOAD)
        API->>FS: Write PDF to papers/{paper_id}.pdf
        API->>DB: INSERT PaperFile (paper_id, path, checksum, size)
        API->>DB: INSERT Job (paper_id, status=QUEUED)
        API->>Queue: Enqueue process_paper(paper_id, job_id)
        API-->>UI: {paper_id, job_id} 201
        UI-->>User: Show JobStatusPoller (polls every 2s)
    end

    Worker->>Queue: Dequeue process_paper task
    Worker->>Worker: Run full pipeline (see §7)
    Worker->>DB: UPDATE Job status=PROCESSED
```

---

## 5. Data Flow: arXiv Ingestion

```mermaid
sequenceDiagram
    actor User
    participant UI as "Next.js /add"
    participant API as "FastAPI /api/papers/arxiv"
    participant arXiv as "arXiv Atom API"
    participant DB as "SQLite catalog.db"
    participant FS as "PDF File Store"
    participant Queue as "Huey Queue"
    participant Worker as "Huey Worker"

    User->>UI: Enter arXiv URL, click Ingest
    UI->>API: POST /api/papers/arxiv {arxiv_url}

    API->>API: Extract arXiv ID\n(e.g. "2301.12345")

    API->>DB: SELECT Paper WHERE arxiv_id = ?
    alt Already ingested by arXiv ID
        DB-->>API: Existing Paper
        API-->>UI: {paper_id (existing)} 200
    else Not found by arXiv ID
        API->>arXiv: GET /api/query?id_list={arxiv_id}
        arXiv-->>API: Atom XML (title, authors, abstract, categories, pdf_url)
        API->>API: Download PDF bytes from\narxiv.org/pdf/{id}
        API->>API: SHA-256 checksum

        API->>DB: SELECT PaperFile WHERE checksum = ?
        alt Duplicate by PDF checksum
            DB-->>API: Existing PaperFile
            API-->>UI: {paper_id (existing)} 200
        else Truly new
            API->>DB: INSERT Paper (source=ARXIV, arxiv_id, metadata)
            API->>DB: INSERT PaperAuthor[] (from Atom feed)
            API->>DB: INSERT Tag[] (from categories, source=ARXIV)
            API->>FS: Write PDF
            API->>DB: INSERT PaperFile
            API->>DB: INSERT Job (status=QUEUED)
            API->>Queue: Enqueue process_paper()
            API-->>UI: {paper_id, job_id} 201
        end
    end

    Worker->>Queue: Dequeue + run pipeline
```

---

## 6. Deduplication Logic

The system applies three independent, layered deduplication checks to avoid reprocessing a paper already in the catalog.

```mermaid
flowchart TD
    Start(["Ingest request received"])

    Start --> CheckSource{Source?}

    CheckSource -- "arXiv" --> CheckArxivID["Check: Paper.arxiv_id\n= extracted ID?"]
    CheckSource -- "PDF Upload" --> CheckChecksum["Check: PaperFile.checksum_sha256\n= SHA-256 of bytes?"]

    CheckArxivID -- "EXISTS" --> ReturnExisting1(["Return existing paper_id\n(HTTP 200)"])
    CheckArxivID -- "NOT EXISTS" --> DownloadPDF["Download PDF bytes\nfrom arXiv"]

    DownloadPDF --> CheckChecksum2["Check: PaperFile.checksum_sha256\n= SHA-256 of downloaded bytes?"]
    CheckChecksum2 -- "EXISTS" --> ReturnExisting2(["Return existing paper_id\n(HTTP 200)"])
    CheckChecksum2 -- "NOT EXISTS" --> CreateNew

    CheckChecksum -- "EXISTS" --> ReturnExisting3(["Return existing paper_id\n(HTTP 200)"])
    CheckChecksum -- "NOT EXISTS" --> CreateNew

    CreateNew["Create Paper + File + Job\nEnqueue process_paper()"]
    CreateNew --> Done(["Return {paper_id, job_id}\n(HTTP 201)"])
```

---

## 7. Worker Pipeline

The `process_paper` Huey task runs these seven sequential stages. Each stage reads from and writes to the shared SQLite database. If any stage raises an exception, the job is marked `FAILED` and the error message stored.

```mermaid
flowchart TD
    Start(["process_paper(paper_id, job_id)\ndequeued from Huey"])

    Start --> GuardCheck{"Job.status\n== PROCESSED?"}
    GuardCheck -- "Yes (idempotent guard)" --> Skip(["Return early, no-op"])
    GuardCheck -- "No" --> Stage1

    Stage1["Stage 1: PARSE\njob.stage = PARSE\n\nPdfExtractor.extract(pdf_bytes)\n→ full_text, title, authors, abstract\n\nWrites: paper_text, updates Paper metadata"]

    Stage1 --> Stage2

    Stage2["Stage 2: CHUNK\njob.stage = CHUNK\n\nchunk_text(full_text, size=1000, overlap=100)\n→ ChunkResult[]\n\nDeletes old PaperChunk rows\nWrites: PaperChunk[] rows"]

    Stage2 --> Stage3

    Stage3["Stage 3: SUMMARIZE\njob.stage = SUMMARIZE\n\nSummarizer.summarize(abstract, full_text)\n→ SummaryResult(tldr, structured_json)\n\nDeletes old PaperSummary\nWrites: PaperSummary row"]

    Stage3 --> Stage4

    Stage4["Stage 4: TAG\njob.stage = TAG\n\nTagger.extract(title, abstract)\n→ TaggerResult(keywords: str[])\n\nDeletes old PaperTag (KEYWORD source)\nUpserts Tag records (unique constraint)\nWrites: PaperTag[] rows (source=KEYWORD)"]

    Stage4 --> Stage5

    Stage5["Stage 5: REF_EXTRACT\njob.stage = REF_EXTRACT\n\nextract_references(full_text)\n→ ExtractedReference[]\n(ref_index, raw_text, parsed_json)\n\nDeletes old PaperReference rows\nWrites: PaperReference[] rows"]

    Stage5 --> Stage6

    Stage6["Stage 6: REF_RESOLVE\njob.stage = REF_RESOLVE\n\nRefResolver.resolve_all(references)\n→ [(PaperReference, ResolvedWork | None)]\n\nFor each resolved work:\n  Upserts Work (by DOI or arXiv ID)\n  Writes ReferenceResolution (status=RESOLVED)\n  Writes Citation (source→work edge)\nFor unresolved:\n  Writes ReferenceResolution (status=UNRESOLVED)"]

    Stage6 --> Stage7

    Stage7["Stage 7: INDEX\n(FTS5 — managed by DB triggers)\nNo explicit stage; triggers fire automatically\non INSERT/UPDATE of papers + paper_text"]

    Stage7 --> Done["job.status = PROCESSED\njob.stage = INDEX\ncommit"]
    Done --> End(["Task complete"])

    Stage1 -- "exception" --> Fail
    Stage2 -- "exception" --> Fail
    Stage3 -- "exception" --> Fail
    Stage4 -- "exception" --> Fail
    Stage5 -- "exception" --> Fail
    Stage6 -- "exception" --> Fail
    Fail["job.status = FAILED\njob.error = str(e)\ncommit; raise"]
    Fail --> End2(["Task ends (will not retry by default)"])
```

### Stage Inputs / Outputs Summary

| Stage | Reads | Writes | External I/O |
|---|---|---|---|
| PARSE | `PaperFile` (PDF bytes) | `paper_text`, `Paper.title/abstract/authors` | None |
| CHUNK | `paper_text.full_text` | `paper_chunks[]` | None |
| SUMMARIZE | `Paper.abstract`, `paper_text.full_text` | `paper_summaries` | None |
| TAG | `Paper.title`, `Paper.abstract` | `tags`, `paper_tags` | None |
| REF_EXTRACT | `paper_text.full_text` | `paper_references[]` | None |
| REF_RESOLVE | `paper_references[]` | `works`, `reference_resolutions`, `citations` | OpenAlex, CrossRef |
| INDEX | _(triggers fire automatically)_ | `paper_search` (FTS5) | None |

---

## 8. Reference Resolution Chain

`RefResolver.resolve_one()` runs three strategies in order, short-circuiting on the first success.

```mermaid
flowchart TD
    Input(["PaperReference\n(ref_index, raw_text, parsed_json)"])

    Input --> S1{"Strategy 1:\nDOI regex\nin raw_text?"}

    S1 -- "DOI found" --> OA1["GET api.openalex.org/works/\nhttps://doi.org/{doi}"]
    OA1 -- "200 OK" --> Return1(["ResolvedWork\nconfidence=1.0\nmethod='doi'\nsource='DOI'"])
    OA1 -- "4xx / 5xx / timeout" --> S2

    S1 -- "No DOI" --> S2

    S2{"Strategy 2:\nTitle in parsed_json?"}

    S2 -- "Title found" --> OA2["GET api.openalex.org/works\n?filter=title.search:{title}"]
    OA2 -- "200 OK, results" --> Sim2{"SequenceMatcher\nratio ≥ 0.85?"}
    Sim2 -- "Yes" --> Return2(["ResolvedWork\nconfidence=ratio\nmethod='openalex_title'"])
    Sim2 -- "No" --> S3
    OA2 -- "error" --> S3

    S2 -- "No title" --> S3

    S3{"Strategy 3:\nCrossRef fallback"}

    S3 --> CR["GET api.crossref.org/works\n?query.bibliographic={raw_text[:200]}"]
    CR -- "200 OK, first result" --> Sim3{"SequenceMatcher\nratio ≥ 0.85?"}
    Sim3 -- "Yes" --> Return3(["ResolvedWork\nconfidence=ratio\nmethod='crossref'"])
    Sim3 -- "No" --> Fail
    CR -- "error" --> Fail

    Fail(["None\n(unresolved reference)"])
```

### Confidence & External Safety

- **Timeout:** 10 seconds for all HTTP calls (`httpx`)
- **User-Agent:** `papers-mvp/1.0 (mailto:research@meitheal.io)` (polite crawl header)
- **Exception handling:** `except Exception` wraps each strategy; a single network failure does not abort the whole pipeline
- **Confidence threshold:** 0.85 — below this, the match is discarded and the next strategy is tried

---

## 9. Full-Text Search Architecture

### FTS5 Table Structure

```mermaid
graph LR
    subgraph VirtualTable["FTS5 Virtual Table: paper_search"]
        PS["paper_id (UNINDEXED)\ntitle\nfull_text\ntokenize='porter ascii'"]
    end

    subgraph Triggers["Auto-sync Triggers"]
        T1["fts_papers_ai\n(papers AFTER INSERT)"]
        T2["fts_papers_au\n(papers AFTER UPDATE title)"]
        T3["fts_papers_ad\n(papers AFTER DELETE)"]
        T4["fts_paper_text_ai\n(paper_text AFTER INSERT)"]
        T5["fts_paper_text_au\n(paper_text AFTER UPDATE)"]
    end

    papers_tbl["papers table"] --> T1 --> PS
    papers_tbl --> T2 --> PS
    papers_tbl --> T3 --> PS
    paper_text_tbl["paper_text table"] --> T4 --> PS
    paper_text_tbl --> T5 --> PS
```

### Search Query Flow

```mermaid
sequenceDiagram
    participant UI as "Next.js"
    participant API as "GET /api/papers?q=..."
    participant FTS as "paper_search (FTS5)"
    participant DB as "papers table"

    UI->>API: GET /api/papers?q=transformer+attention&page=1

    API->>FTS: SELECT paper_id, bm25(paper_search)\nFROM paper_search\nWHERE paper_search MATCH 'transformer attention'\nORDER BY bm25(paper_search)

    alt Valid FTS query
        FTS-->>API: Ranked paper_id list
        API->>DB: SELECT * FROM papers WHERE id IN (...)\npreserving BM25 order
        DB-->>API: Paper rows + authors + tags
        API-->>UI: PaperListResponse (ranked results)
    else Malformed query (e.g. bare AND)
        FTS-->>API: OperationalError
        API->>DB: SELECT * FROM papers\nORDER BY created_at DESC
        DB-->>API: All papers
        API-->>UI: PaperListResponse (fallback, unranked)
    end
```

### Supported Filter Parameters

| Parameter | Search mechanism | Example |
|---|---|---|
| `q` | FTS5 MATCH (BM25 ranked) | `?q=neural+network` |
| `tag` | `paper_tags.tag = ?` join | `?tag=machine-learning` |
| `author` | `paper_authors.name LIKE ?` | `?author=LeCun` |
| `year` | `papers.published_year = ?` | `?year=2024` |

---

## 10. Database Schema (ER Diagram)

```mermaid
erDiagram
    papers {
        UUID id PK
        string source "UPLOAD | ARXIV"
        string title
        string abstract
        string arxiv_id "UNIQUE, nullable"
        int published_year
        datetime created_at
    }

    paper_authors {
        int id PK
        UUID paper_id FK
        int author_order "UNIQUE per paper"
        string name
    }

    paper_files {
        UUID id PK
        UUID paper_id FK
        string storage_path
        string checksum_sha256 "UNIQUE"
        int size_bytes
    }

    jobs {
        UUID id PK
        UUID paper_id FK
        string status "QUEUED|RUNNING|PROCESSED|FAILED"
        string stage "FETCH|PARSE|CHUNK|SUMMARIZE|TAG|REF_EXTRACT|REF_RESOLVE|INDEX"
        string error "nullable"
        datetime created_at
        datetime updated_at
    }

    paper_text {
        int id PK
        UUID paper_id FK "UNIQUE"
        text full_text
    }

    paper_chunks {
        int id PK
        UUID paper_id FK
        int chunk_index
        text text
        int start_offset
        int end_offset
    }

    paper_summaries {
        int id PK
        UUID paper_id FK "UNIQUE"
        text tldr
        text structured_json
        string model
        datetime created_at
    }

    tags {
        int id PK
        string tag "UNIQUE"
    }

    paper_tags {
        int id PK
        UUID paper_id FK
        int tag_id FK
        string source "KEYWORD|ARXIV|LLM"
    }

    paper_references {
        int id PK
        UUID paper_id FK
        int ref_index
        text raw_text
        text parsed_json "nullable"
    }

    works {
        int id PK
        string doi "UNIQUE, nullable"
        string arxiv_id "UNIQUE, nullable"
        string title
        int year
        string venue
        text authors_json
        string source "DOI|ARXIV|TITLE_SEARCH"
    }

    reference_resolutions {
        int id PK
        int reference_id FK "UNIQUE"
        int work_id FK
        float confidence
        string method
        string status "RESOLVED|UNRESOLVED"
    }

    citations {
        int id PK
        UUID source_paper_id FK
        int target_work_id FK
        UUID target_paper_id FK "nullable"
        datetime created_at
    }

    papers ||--o{ paper_authors : "has"
    papers ||--o{ paper_files : "has"
    papers ||--o{ jobs : "has"
    papers ||--o| paper_text : "has"
    papers ||--o{ paper_chunks : "has"
    papers ||--o| paper_summaries : "has"
    papers ||--o{ paper_tags : "has"
    papers ||--o{ paper_references : "has"
    papers ||--o{ citations : "cites (source)"
    papers ||--o{ citations : "cited by (target)"
    tags ||--o{ paper_tags : "used by"
    paper_references ||--o| reference_resolutions : "resolved to"
    works ||--o{ reference_resolutions : "resolves"
    works ||--o{ citations : "target work"
```

### Table Counts at a Glance

| Table | Growth driver |
|---|---|
| `papers` | 1 row per unique ingested paper |
| `paper_authors` | ~3-8 rows per paper |
| `paper_files` | 1 row per paper (one PDF) |
| `jobs` | 1+ per paper (retries create new rows) |
| `paper_text` | 1 row per paper (after PARSE) |
| `paper_chunks` | ~50-200 rows per paper (1000-char chunks) |
| `paper_summaries` | 1 row per paper |
| `paper_tags` | ~10 KEYWORD + N ARXIV tags per paper |
| `paper_references` | ~20-50 rows per paper |
| `works` | Shared across all papers (deduplicated by DOI) |
| `reference_resolutions` | 1 per reference |
| `citations` | 1 per resolved reference |

---

## 11. Frontend Architecture

### Component Tree

```mermaid
graph TD
    Root["layout.tsx\n(Root Layout)\nHeader nav: '/' and '/add'"]

    Root --> Home["page.tsx\n(Home / Library)"]
    Root --> Add["add/page.tsx\n(Client Component)\nUpload + arXiv tabs"]
    Root --> Jobs["jobs/page.tsx\n(Server Component)\nJob history table"]
    Root --> PaperDetail["papers/[id]/page.tsx\n(Server Component, async)\nFull paper view"]

    Add --> Poller["JobStatusPoller.tsx\n(Client Component)\nPolls getJobStatus()\nevery 2s"]

    Home --> API1["(planned) listPapers()"]
    Add --> API2["uploadPaper()\ningestArxiv()"]
    Jobs --> API3["listJobs()\nauto-refresh 5s\nif QUEUED/RUNNING"]
    PaperDetail --> API4["getPaperDetail()\ngetPaperReferences()\ngetPaperCitations()\n(Promise.allSettled)"]
    Poller --> API5["getJobStatus()\nevery 2s"]
```

### Frontend Data Fetching Patterns

```mermaid
sequenceDiagram
    participant Browser
    participant RSC as "Server Component\n(Node.js)"
    participant CC as "Client Component\n(Browser)"
    participant API as "FastAPI"

    Note over RSC,API: Paper Detail Page (RSC — no client JS for initial data)
    Browser->>RSC: GET /papers/{id}
    RSC->>API: getPaperDetail(id) [parallel with...]
    RSC->>API: getPaperReferences(id)
    RSC->>API: getPaperCitations(id)
    API-->>RSC: PaperDetail + Refs + Citations
    RSC-->>Browser: Fully rendered HTML

    Note over CC,API: Job Status Polling (Client Component)
    Browser->>CC: Render JobStatusPoller
    loop every 2 seconds until PROCESSED or FAILED
        CC->>API: GET /api/jobs/{job_id}
        API-->>CC: JobResponse {status, stage}
        CC->>Browser: Update status badge
    end
    CC->>Browser: Show link to /papers/{paper_id}
```

### Page Responsibilities

| Route | Rendering | Primary API Calls | Notes |
|---|---|---|---|
| `/` | Server Component | _(library index, planned)_ | Stub in MVP |
| `/add` | Client Component | `POST /api/papers/upload`, `POST /api/papers/arxiv` | Rate-limited 10/min |
| `/jobs` | Server Component | `GET /api/jobs` | Auto-refreshes if jobs are active |
| `/papers/[id]` | Server Component | `GET /api/papers/{id}`, `/references`, `/citations` | `notFound()` on fetch error |

---

## 12. Configuration & Environment

```mermaid
graph LR
    subgraph EnvFile[".env / environment"]
        E1["DATABASE_URL\n(sqlite+aiosqlite:///./data/catalog.db)"]
        E2["HUEY_DB_PATH\n(./data/huey.db)"]
        E3["STORAGE_DIR\n(./data/papers)"]
        E4["LLM_API_KEY\n(unused in MVP)"]
        E5["ARXIV_API_BASE_URL\n(https://export.arxiv.org/api/query)"]
        E6["DEBUG\n(false)"]
        E7["CORS_ORIGINS\n(JSON list of allowed browser origins)\ndefault: localhost:3000/3001/3002"]
    end

    subgraph Settings["src/config.py — Settings (Pydantic BaseSettings)"]
        S["Settings singleton\n(settings = Settings())"]
    end

    subgraph Consumers["Consumers"]
        C1["src/db/session.py\n→ async engine URL"]
        C2["src/worker/huey_app.py\n→ SqliteHuey filename"]
        C3["src/services/storage.py\n→ LocalStorageBackend path"]
        C4["src/api/arxiv.py\n→ arXiv base URL"]
        C5["src/main.py\n→ CORSMiddleware allow_origins"]
    end

    EnvFile --> Settings
    S --> C1
    S --> C2
    S --> C3
    S --> C4
    S --> C5
```

---

## 13. Inter-Component Data Contracts

This section documents the exact shapes passed between components at each boundary.

### Upload API → Worker

```mermaid
graph LR
    API["POST /api/papers/upload"] -- "huey enqueue" --> Q["SqliteHuey\n'meitheal' queue"]
    Q -- "dequeue" --> W["process_paper(\n  paper_id: str,\n  job_id: str\n)"]
```

The only data passed over the queue is `paper_id` and `job_id` (both UUIDs as strings). The worker fetches all state it needs from the database — there is no large payload in the queue.

### PdfExtractor → Worker (extracted data)

```python
ExtractResult(
    full_text: str,
    title: str | None,
    authors: list[str],
    abstract: str | None,
)
```

### Chunker → Worker (chunk records)

```python
ChunkResult(
    chunk_index: int,
    text: str,
    start_offset: int,
    end_offset: int,
)
```

### Summarizer → Worker (summary record)

```python
SummaryResult(
    tldr: str,
    structured_json: str,  # JSON: {source_type, preview}
)
```

### Tagger → Worker (keyword list)

```python
TaggerResult(
    keywords: list[str],  # top-10, stop-words removed
)
```

### RefExtractor → Worker (raw references)

```python
ExtractedReference(
    ref_index: int,
    raw_text: str,
    parsed_json: str | None,  # JSON: {authors, title, year, venue}
)
```

### RefResolver → Worker (resolved works)

```python
ResolvedWork(
    title: str,
    doi: str | None,
    arxiv_id: str | None,
    year: int | None,
    authors: list[str],
    source: WorkSource,   # DOI | ARXIV | TITLE_SEARCH
    method: str,          # "doi" | "openalex_title" | "crossref"
    confidence: float,    # 0.0 – 1.0
)
```

### FastAPI → Frontend (key response shapes)

```mermaid
graph LR
    subgraph FastAPI["FastAPI Responses (JSON)"]
        PL["PaperListItem\n{id, title, abstract,\npublished_year, arxiv_id,\nsource, created_at,\nauthors[], tags[]}"]
        PD["PaperDetail\n(extends PaperListItem)\n+ tldr, structured_json,\nfull_text, status"]
        RS["ReferenceSchema\n{id, raw_text, parsed_json,\nref_index, resolved_work_id,\nresolved_doi, resolved_arxiv_id,\nresolved_title}"]
        CS["CitationSchema\n{id, source_paper_id,\nsource_title, created_at}"]
        JR["JobResponse\n{id, paper_id, status, stage,\nerror, created_at, updated_at}"]
        MR["MetricsResponse\n{papers_total,\njobs: {queued, running,\n  processed, failed, total},\nresolution: {total_references,\n  resolved, unresolved,\n  resolution_rate}}"]
    end
```

---

## Appendix: Technology Stack

| Layer | Technology | Version | Purpose |
|---|---|---|---|
| Frontend | Next.js | 16.1.6 | React Server Components + Turbopack |
| Frontend | React | 19 | UI rendering |
| Frontend | TypeScript | 5 | Type safety |
| Frontend | Tailwind CSS | 4.0 | Styling |
| Backend | FastAPI | 0.115.12 | REST API framework |
| Backend | Pydantic | v2 | Schema validation |
| Backend | SQLAlchemy | (async) | ORM with aiosqlite |
| Backend | Alembic | — | Schema migrations |
| Backend | Huey | 2.5.3 | Task queue (SQLite backend) |
| Backend | PyMuPDF (fitz) | — | PDF text extraction |
| Backend | httpx | — | Async HTTP client for external APIs |
| Backend | slowapi | — | Rate limiting |
| Database | SQLite | 3 (WAL mode) | Primary database + FTS5 |
| Containerization | Docker | — | Production packaging |
| Test runner (backend) | pytest + pytest-asyncio | 8.3.5 / 0.24.0 | Backend unit + integration tests |
| Test runner (frontend) | Vitest | 4.x | Frontend unit + component tests |
| E2E tests | Playwright | 1.58 | Frontend end-to-end |
