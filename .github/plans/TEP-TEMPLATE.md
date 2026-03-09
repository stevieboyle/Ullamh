# Technical Execution Plan — [STORY-ID]: [Story Title]

**Story:** [STORY-ID]  
**Author:** [@agent-name]  
**Status:** Draft | Approved | In Progress | Complete  
**Architect Review:** [ ] Pending | [ ] LGTM (@Architect, date)  
**Date:** YYYY-MM-DD

---

## 1. Objective

One paragraph describing what this story delivers and why it matters to the system. Reference the relevant Architecture Spec section.

> Example: "This TEP covers implementing `POST /api/items` (§5.2 of the Architecture Specification). It introduces the ingestion path, queuing an async task that fetches metadata, stores a record, and triggers the processing pipeline."

---

## 2. Library & Dependency Versions

List every new or upgraded dependency this story requires.

| Package | Version | Purpose |
|---|---|---|
| `example-lib` | `1.2.3` | Brief description |

> If no new dependencies are required, state: "No new dependencies."

---

## 3. Target Files

List every file that will be **created** or **modified**. Be explicit.

| Action | File Path | Change Summary |
|---|---|---|
| Create | `src/api/ingest.py` | New route module for item ingestion |
| Modify | `src/worker/tasks.py` | Add `process_item` async task |
| Modify | `tests/api/test_ingest.py` | New test file |

---

## 4. Pydantic Model Schemas

Define or reference the exact Pydantic models involved. For new models, show the full field set with types. For modified models, show the delta.

```python
class IngestRequest(BaseModel):
    item_id: str = Field(..., pattern=r"^[A-Za-z0-9._-]+$")

class IngestResponse(BaseModel):
    job_id: int
    item_id: int | None = None
    status: JobStatus
```

---

## 5. API Signatures

List every endpoint created or modified.

```
POST /api/items
  Request:  IngestRequest
  Response: IngestResponse (202)
  Errors:   422 (validation), 429 (rate limit), 409 (duplicate)
```

---

## 6. Task & Worker Signatures

List every async task created or modified.

```python
@task_queue.task()
def process_item(job_id: int, item_id: str) -> None: ...
```

---

## 7. Cross-Cutting Concerns

**This section is mandatory.** For each concern, state the impact or explicitly confirm it is N/A with justification.

| Concern | Impact / Handling |
|---|---|
| **CORS** | Does this story add new origins or change CORS policy? List any `allow_origins` changes needed. N/A if no frontend-facing endpoint is added. |
| **Authentication / Authorization** | Is this endpoint protected? How? (Currently: no auth for MVP. Note if auth will be added.) |
| **Rate Limiting** | Does this story add or modify rate-limited endpoints? If so, name the `RateLimiter` instance and limit values. Confirm test fixture resets it. |
| **Logging / Observability** | What is logged at each stage (request received, task queued, task complete/failed)? |
| **Error Propagation** | How do task failures surface to the job status model? What `JobStatus` enum value is set on failure? |
| **Idempotency** | How is duplicate detection handled? (unique key, checksum, or other?) |

---

## 8. Acceptance Criteria (from Jira)

Copy the ACs directly from the Jira story. Each must be:
- Specific and testable
- Reference exact model/route/enum names
- Include at least one boundary-condition AC

- [ ] AC 1 — ...
- [ ] AC 2 — ...
- [ ] AC 3 — ...

---

## 9. Test Plan

| Test File | Test Type | What It Covers |
|---|---|---|
| `tests/api/test_ingest.py` | Unit/Integration (pytest) | Happy path, duplicate detection, 422 validation |
| `web/tests/smoke.spec.ts` | E2E (live stack) | Submit item via UI, job status updates to COMPLETE |

**Live-stack test required:** [ ] Yes — list which test exercises the live HTTP stack | [ ] N/A — justify

---

## 10. Rollback Plan

If this change must be reverted:
1. Step 1 — ...
2. Step 2 — ...

---

## 11. Open Questions / Risks

| # | Question | Owner | Status |
|---|---|---|---|
| 1 | Example question | @agent | Open |

---

## Architect Sign-off

> @Architect: Please review sections 3 (Target Files), 5 (API Signatures), and 7 (Cross-Cutting Concerns) before approving.

- [ ] **LGTM** — approved to proceed
- [ ] **Changes requested** — (comments below)
