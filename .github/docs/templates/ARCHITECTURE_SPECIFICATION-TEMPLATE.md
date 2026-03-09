# Architecture Specification — [PROJECT_NAME]
**Jira Epic:** [EPIC_ID]  
**Status:** Draft (v0.1)  
**Last updated:** [DATE]

> **Usage:** Fill in this document before coding begins. It is the single source of truth for all implementation decisions. All TEPs and story refinements must cite specific sections of this spec. When the MVP is delivered, promote Status to "v1.0 (As-Built)" and document what changed in the Change Log.

---

## 1. Purpose

Describe what the MVP will build. Use bullet points for the key capabilities.

- [Capability 1]
- [Capability 2]
- [Capability 3]

**Non-goals for MVP:** List explicitly what will NOT be built in this iteration (important for scope control).

- [Non-goal 1]
- [Non-goal 2]

---

## 2. Tech Stack

### Frontend
- **[Framework]** — [language/version], [router type]
- **UI:** [Component library or CSS approach]
- **Data fetching:** [e.g. fetch, React Query]

### Backend
- **[Framework]** — [language/version]
- **ORM:** [e.g. SQLAlchemy] — allows future migration from [DB A] to [DB B]
- **Validation:** [e.g. Pydantic v2]
- **Auth:** [e.g. No-auth for MVP / JWT / Session]

### Async Processing
- **[Task queue]** — [broker type, e.g. SQLite-backed / Redis-backed]

### Storage
- **[Primary DB]** — [version/edition]
  - [Special features, e.g. FTS5 for search, pgvector for embeddings]
- **[File storage]** — [e.g. local filesystem, S3] — [abstraction approach for future swap]

### External Services
- [Service 1] — [purpose]
- [Service 2] — [purpose]

### Migration Path (Post-MVP)
- [Current DB] → [Target DB] when [scaling trigger]
- [Other planned migrations or upgrades]

---

## 3. Data Model

### Core Entities

Describe the primary domain objects and their relationships.

| Entity | Key Fields | Relationships |
|---|---|---|
| `[Entity1]` | id, [field], [field], created_at | has many [Entity2] |
| `[Entity2]` | id, [entity1]_id, [field], status | belongs to [Entity1] |

### Status / State Enums

```python
class [StatusEnum](str, Enum):
    [VALUE_1] = "[value_1]"    # [meaning]
    [VALUE_2] = "[value_2]"    # [meaning]
    COMPLETE  = "COMPLETE"
    FAILED    = "FAILED"
```

---

## 4. API Design

### Base URL
`[BASE_URL]/api/[version_prefix]`

### Endpoints

| Verb | Path | Request Body | Response | Description |
|---|---|---|---|---|
| POST | `/[resource]` | `[RequestSchema]` | `[ResponseSchema]` (202) | [What it does] |
| GET | `/[resource]` | — | `List[[ResponseSchema]]` (200) | [What it returns] |
| GET | `/[resource]/{id}` | — | `[ResponseSchema]` (200) | [What it returns] |

### Common Error Responses
- `422` — Validation error (Pydantic)
- `429` — Rate limit exceeded
- `409` — Conflict (duplicate record)
- `404` — Not found

---

## 5. Processing Pipeline

Describe the async task chain in order.

| Stage | Task Name | Input | Output | Service |
|---|---|---|---|---|
| 1 | `[task_1]` | [input] | [output] | `src/services/[service].py` |
| 2 | `[task_2]` | [input] | [output] | `src/services/[service].py` |
| ... | | | | |

### Stage Status Transitions
```
QUEUED → [STAGE_1] → [STAGE_2] → ... → COMPLETE
                                      ↘ FAILED (any stage)
```

---

## 6. Security

| Concern | Approach |
|---|---|
| Authentication | [e.g. No-auth for MVP; JWT post-MVP] |
| CORS | [Allowed origins; CORSMiddleware config] |
| Rate limiting | [Limits per endpoint; library used] |
| File upload safety | [Max size, type validation, storage isolation] |
| Dependency safety | [e.g. `pip audit` / `npm audit` in CI] |

---

## 7. Storage & File Handling

- **Upload path:** [Where files land after upload]
- **Naming convention:** [e.g. UUID-named to prevent path traversal]
- **Abstraction:** [e.g. StorageBackend ABC with LocalStorage / S3Storage implementations]
- **Future swap path:** [How to move from local filesystem to cloud storage]

---

## 8. Search Architecture

- **Search mechanism:** [e.g. SQLite FTS5 / Postgres full-text / Elasticsearch]
- **Indexed fields:** [Which fields are searchable]
- **Query format:** [e.g. keyword, boolean, phrase]
- **Post-MVP:** [e.g. Vector embeddings with pgvector]

---

## 9. Frontend Pages & Components

| Route | Component | Data Source | Notes |
|---|---|---|---|
| `/` | `[Component]` | `GET /api/[resource]` | [Server/Client component?] |
| `/[resource]/[id]` | `[Component]` | `GET /api/[resource]/{id}` | [Notes] |
| `/[action]` | `[Component]` | POST on submit | [Notes] |

**Async state component:** `[ComponentName]` — polls `GET /api/jobs/{id}` every [N] seconds.

---

## 10. Testing Strategy

| Layer | Tool | Location | Coverage Target |
|---|---|---|---|
| Backend unit/integration | pytest + pytest-asyncio | `tests/` | [Target %] |
| Frontend unit | [e.g. Vitest] | `web/src/` | [Target %] |
| E2E / live-stack | Playwright | `web/tests/` | Key user journeys |

**Rate limiter test hygiene:** All `RateLimiter` instances must be reset in `conftest.py` between tests.

---

## 11. Performance Targets

| Operation | Target | Notes |
|---|---|---|
| API response (p95) | < [X] ms | Under [Y] concurrent users |
| Full pipeline for [input type] | < [X] seconds | For [typical input size] |
| Search query | < [X] ms | For [corpus size] |

---

## 12. Non-Functional Requirements

- **Idempotency:** All pipeline tasks must be safe to re-run.
- **Data integrity:** [Constraint, e.g. checksum deduplication before INSERT]
- **Observability:** [Log points, health endpoint, metrics endpoint]
- **Deployment:** [Target environment, e.g. Docker Compose, single-host]

---

## 13. Open Decisions

> Items here must be resolved before a story moves to In Progress. Link each to a spike story.

| # | Decision | Owner | Spike | Status |
|---|---|---|---|---|
| 1 | [Decision to make] | @[agent] | [STORY-ID] | Open / Resolved |

---

## Change Log

| Version | Date | Summary |
|---|---|---|
| v0.1 | [DATE] | Initial draft |
| v1.0 | [DATE] | As-built; promoted from draft after MVP delivery |
