# Architecture Guide — [PROJECT_NAME] ([EPIC_ID])

**Author:** @Architect  
**Version:** 1.0  
**Date:** [DATE]

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Deployment Architecture](#2-deployment-architecture)
3. [API Surface Reference](#3-api-surface-reference)
4. [Key Data Flows](#4-key-data-flows)
5. [Deduplication / Idempotency Logic](#5-deduplication--idempotency-logic)
6. [Worker / Async Pipeline](#6-worker--async-pipeline)
7. [Database Schema](#7-database-schema)
8. [Frontend Architecture](#8-frontend-architecture)
9. [Configuration & Environment](#9-configuration--environment)
10. [Inter-Component Data Contracts](#10-inter-component-data-contracts)

---

## 1. System Overview

Describe the system in 2–3 sentences. Include the number of tiers, the primary user journey, and the key moving parts.

```
[Draw a simple ASCII or Mermaid component diagram]

┌─────────────────────────────────────────────────────────┐
│  [COMPONENT 1]   [COMPONENT 2]   [COMPONENT 3]          │
│       │               │               │                  │
│       └───────────────┴───────────────┘                  │
│                       │                                  │
│                 Shared data store                        │
└─────────────────────────────────────────────────────────┘
```

---

## 2. Deployment Architecture

### Container / Process Layout

```
[Draw the deployment layout — Docker services, processes, ports]

Service: [api]         → Port: [8000]    → [What it runs]
Service: [worker]      → Port: n/a       → [What it runs]
Service: [frontend]    → Port: [3000]    → [What it runs]

Shared volumes:
  ./data/ → [api], [worker]
```

### Local Dev Layout

```
[Show the local dev file structure]

project-root/
  venv/             ← Python virtual environment
  src/              ← Backend source
  web/              ← Frontend source
  data/             ← Runtime data (SQLite DB, uploads, etc.)
  tests/            ← Backend test suite
  alembic/          ← DB migrations
```

---

## 3. API Surface Reference

List every API endpoint grouped by resource.

```
[RESOURCE 1]
  [VERB] /api/[path]        [Brief description]
  [VERB] /api/[path]/{id}   [Brief description]

[RESOURCE 2]
  [VERB] /api/[path]        [Brief description]

Middleware:
  [Middleware 1]  — [What it does, e.g. "CORSMiddleware — allows listed origins"]
  [Middleware 2]  — [What it does, e.g. "RateLimitMiddleware — 10 req/60s per endpoint"]
```

---

## 4. Key Data Flows

For each major user action, show the request path through the system.

### 4.1 [Primary Ingestion Flow, e.g. File Upload]

```
[ACTOR] → [LAYER 1] → [LAYER 2] → [LAYER 3] → [STORAGE]
```

### 4.2 [Secondary Flow, e.g. Background Processing]

```
[TRIGGER] → [WORKER] → [SERVICES...] → [DATABASE]
```

---

## 5. Deduplication / Idempotency Logic

Describe how the system prevents duplicate records and ensures tasks are safe to re-run.

```
Deduplication check order:
  1. [Check 1, e.g. SHA-256 checksum of input]
  2. [Check 2, e.g. External ID (arXiv ID, DOI)]
  3. [If duplicate found → response: 409 Conflict + existing record]
```

---

## 6. Worker / Async Pipeline

Document the async task chain, stage names, and status transitions.

```
Stage sequence:
  [STAGE_1] → [STAGE_2] → [STAGE_3] → ... → COMPLETE
                                            ↘ FAILED

Status enum values:
  [STATUS_1]   — [meaning]
  [STATUS_2]   — [meaning]
  COMPLETE     — all stages finished successfully
  FAILED       — unrecoverable error; see job.error_detail
```

---

## 7. Database Schema (ER Diagram)

```
[TABLE_1]                    [TABLE_2]
  id: INTEGER (PK)             id: INTEGER (PK)
  [field]: [TYPE]              [field]: [TYPE]
  created_at: DATETIME         [foreign_key]_id → [TABLE_1].id

[TABLE_3] (FTS virtual table, if applicable)
  [indexed_field_1]
  [indexed_field_2]
```

---

## 8. Frontend Architecture

Describe the frontend framework, component types, and data fetching patterns.

```
Routing: [e.g. App Router / Pages Router]
Component strategy: [e.g. Server Components by default; Client Components for interactive leaves]
Data fetching: [e.g. fetch() in Server Components; React Query for polling]

Page structure:
  /               → [What it shows]
  /[resource]     → [List page]
  /[resource]/[id] → [Detail page]
  /[action]       → [e.g. Add / Create page]
```

---

## 9. Configuration & Environment

Document all environment variables.

```
Variable               Default                  Required  Notes
──────────────────────────────────────────────────────────────────
[VAR_1]               [default value]           Yes       [description]
[VAR_2]               [default value]           No        [description]
[VAR_3]               (empty)                   No        [Optional feature — disabled if unset]

Consumers:
  [VAR_1] → [file that reads it] → [what it controls]
```

---

## 10. Inter-Component Data Contracts

Document the key schemas/types that cross component boundaries.

```python
# [Contract 1: e.g. API response schema]
class [ResourceResponse](BaseModel):
    id: int
    [field]: [type]
    status: [StatusEnum]

# [Contract 2: e.g. Task signature]
@[queue].task()
def [task_name]([param]: [type]) -> None: ...
```

---

## Change Log

| Version | Date | Author | Summary |
|---|---|---|---|
| 1.0 | [DATE] | @Architect | Initial as-built guide |
