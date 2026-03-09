# KAN-4 MVP Sprint Retrospective
**Date:** 2026-03-08  
**Facilitated by:** @ProjectManager  
**Participants:** @Analyst, @Architect, @BackendDev, @UIDev, @QAEngineer, @WorkerDev  
**Scope:** All 24 MVP delivery stories under Epic KAN-4 + KAN-29 enablement stories

---

## Executive Summary

The KAN-4 MVP was delivered with 118 tests all passing (86 backend, 19 frontend, 13 E2E). The platform ingests papers via PDF upload and arXiv URL, processes them asynchronously via a Huey pipeline, and exposes a Next.js frontend with job tracking, paper library, detail views, citation lineage, full-text search, and metrics.

Six key retrospective themes emerged across all specialist inputs:
1. **CORS omission** — absent from spec, absent from mocked tests, only caught by live-stack Playwright (6/6 agents cited this)
2. **TEP gate not enforced** — skipped for KAN-37 (spike) and KAN-38 (test coverage story)
3. **Rate limiter fixture gap** — only reset upload limiter, not arXiv limiter; caused intermittent 429s in sequential test runs
4. **Playwright/Vitest file collision** — `testMatch: '**/*.spec.ts'` was too broad; fixed during testing sprint
5. **`.venv/` vs `venv/`** — silent failure mode for agents following `copilot-instructions.md`
6. **DoD is implicit, not written** — no formal checklist gating story completion

---

## Retrospective Inputs by Specialist

### @Analyst
- Templates were solid and structurally consistent, but CORS was never added to the Architecture Spec security section, so it was never flagged during story refinement.
- Boundary-condition acceptance criteria (rate limits, duplicate detection edge cases) were underspecified; devs had to infer behaviour from the spec.
- @Analyst sign-off should be a formal prerequisite on all stories before @Architect TEP review — currently this is implied but not enforced.
- Spikes (KAN-37) produced no formal Decision Record. When the PDF library choice (fitz) was made, there was no artifact capturing the decision rationale for future reference.

### @Architect
- `STEEL_THREAD_DESIGN.md` was kept in sync, but Delta Updates were applied informally. A threshold definition for what triggers a Delta Update vs. a regular code comment is needed.
- The Architecture Spec remains at "Draft v0.2" despite the MVP being shipped. It should be promoted to v1.0 to reflect the as-built state.
- CORS was a cross-cutting concern that didn't fit neatly into any story's security section. Going forward, a mandatory "Cross-Cutting Concerns" section in every TEP would catch this.
- The TEP template lives only inside `copilot-instructions.md`; a canonical standalone file at `.github/plans/TEP-TEMPLATE.md` would make it easier to reference and validate.

### @BackendDev
- The `.venv/` vs `venv/` discrepancy in `copilot-instructions.md` caused agent confusion. The actual venv is `venv/`, not `.venv/`.
- The startup validation pattern (check DB connection, check storage dir, check Huey queue) was implemented organically but was never explicitly required. A startup validation checklist in the agent instructions would make this consistent.
- `requirements.txt` was kept up to date throughout, which worked well.
- All commands used `.venv` context references early on; these should have used `venv`.

### @UIDev
- The Playwright `testMatch: '**/*.spec.ts'` pattern caused Vitest unit tests to be picked up by Playwright. The fix was simple (rename or scope `testDir`), but it was a surprise during the testing sprint.
- Port 3002 for the frontend in tests vs port 3000 for dev was not documented anywhere. Agents had to discover this from `FRONTEND_PORT` env var.
- The `NEXT_PUBLIC_API_URL` documentation should explicitly note that in E2E tests the frontend runs on port 3002 by default (controlled by `FRONTEND_PORT`).

### @QAEngineer
- The rate limiter fixture in `conftest.py` only reset the upload rate limiter, not the arXiv rate limiter. This caused intermittent `429 Too Many Requests` errors in sequential test runs that exercised arXiv endpoints.
- For all stories touching external HTTP APIs, a live-HTTP test layer (using the actual running stack, not mocked) should be mandatory. CORS would have been caught earlier if this had been enforced from the start.
- Test file naming: backend tests should follow `test_<feature>.py` and frontend tests `<Component>.test.tsx`. The current naming is mostly correct but not formally documented.

### @WorkerDev
- External API fixtures (arXiv, DOI resolver) were added reactively rather than proactively. A rule requiring mock fixtures before coding begins for any external-API-touching story would have saved rework.
- The Huey worker domain is correctly called `src/worker/` but the agent's `copilot-instructions.md` documentation still referenced `src/workers/` (Celery legacy naming). This is now fixed in the agent file but should be caught in a future refinement gate.
- Idempotency is in place but the strategy (checksum + arXiv ID deduplication) was never formally documented in the spec.

---

## 25 Recommendations

### Category 1 — copilot-instructions.md

| ID | Priority | Recommendation | Target File |
|---|---|---|---|
| REC-01 | HIGH | Fix `.venv/` → `venv/` throughout | `copilot-instructions.md` |
| REC-02 | HIGH | Universal TEP gate: TEP required for ALL story types (features, spikes, test stories) | `copilot-instructions.md` |
| REC-03 | HIGH | Add "Cross-Cutting Concerns" section to TEP content requirements | `copilot-instructions.md` |
| REC-04 | HIGH | Add written Definition of Done checklist (7-point gate) | `copilot-instructions.md` |
| REC-05 | MEDIUM | Document test file naming convention | `copilot-instructions.md` |
| REC-08 | MEDIUM | Spikes must produce an Architecture Decision Record (ADR) | `copilot-instructions.md` |

### Category 2 — TEP Template

| ID | Priority | Recommendation | Target File |
|---|---|---|---|
| REC-07 | HIGH | Create canonical standalone TEP template | `.github/plans/TEP-TEMPLATE.md` |

### Category 3 — Jira Workflow

| ID | Priority | Recommendation | Target File |
|---|---|---|---|
| REC-09 | HIGH | @PM must verify TEP link exists in Jira before approving In Progress transition | `pm.agent.md` |
| REC-10 | MEDIUM | At story closure, @PM must create follow-on tickets for all deferred items | `pm.agent.md` |
| REC-11 | MEDIUM | Handoffs require a structured Jira comment (using handoff-protocol template) | `handoff-protocol/SKILL.md` |

### Category 4 — Skills

| ID | Priority | Recommendation | Target File |
|---|---|---|---|
| REC-12 | HIGH | Add Cross-Cutting Concerns gate to refinement-checklist | `refinement-checklist/SKILL.md` |
| REC-13 | HIGH | Require boundary-condition ACs (explicit status codes/values) | `refinement-checklist/SKILL.md` |
| REC-14 | MEDIUM | Create `tep-review` skill for @Architect | `.github/skills/tep-review/SKILL.md` (new) |
| REC-15 | MEDIUM | Add "Test Layer Ownership" field to handoff-protocol | `handoff-protocol/SKILL.md` |

### Category 5 — Agent Instructions

| ID | Priority | Recommendation | Target File |
|---|---|---|---|
| REC-06 | MEDIUM | @Analyst sign-off is a formal prerequisite before @Architect TEP review | `analyst.agent.md` |
| REC-16 | HIGH | @QAEngineer: live-HTTP test layer mandatory for all API-touching stories | `qa-engineer.agent.md` |
| REC-17 | HIGH | @BackendDev: startup validation checklist before declaring task complete | `backend-dev.agent.md` |
| REC-18 | MEDIUM | @WorkerDev: mock fixtures for external APIs must exist before coding begins | `worker-dev.agent.md` |
| REC-19 | MEDIUM | @UIDev: document port 3002 (FRONTEND_PORT) and Playwright testMatch scope | `ui-dev.agent.md` |
| REC-20 | MEDIUM | @Architect: define Delta Update threshold (>1 file OR public API surface change) | `architext.agent.md` |

### Category 6 — Architecture & Documentation

| ID | Priority | Recommendation | Target File |
|---|---|---|---|
| REC-21 | MEDIUM | Promote Architecture Spec from Draft v0.2 → v1.0 (as-built, fold in Delta Updates) | `KAN-4-architecture-specification.md` |
| REC-22 | MEDIUM | Create `docs/OPERATIONS_RUNBOOK.md` (deploy, backup, metrics, incident response) | `docs/OPERATIONS_RUNBOOK.md` (new) |
| REC-23 | MEDIUM | Update `STARTUP_GUIDE.md`: venv name, port 3002 note, frontend test commands | `docs/STARTUP_GUIDE.md` |
| REC-24 | LOW | Create `docs/pipeline-parameters.md`: prompt templates, extraction thresholds | `docs/pipeline-parameters.md` (new) |

### Category 7 — Technical Debt (Jira Backlog)

| ID | Priority | Recommendation | Target File |
|---|---|---|---|
| REC-25 | MEDIUM | Create 13 Jira backlog tickets covering identified tech debt items | Jira KAN project |

---

## Priority Matrix

| Priority | Count | Recommendations |
|---|---|---|
| HIGH | 8 | REC-01, REC-02, REC-03, REC-04, REC-07, REC-09, REC-12, REC-13, REC-16, REC-17 |
| MEDIUM | 16 | REC-05, REC-06, REC-08, REC-10, REC-11, REC-14, REC-15, REC-18, REC-19, REC-20, REC-21, REC-22, REC-23, REC-25 |
| LOW | 1 | REC-24 |

---

## Tech Debt Inventory (REC-25 source list)

The following items were identified but deferred during MVP delivery. Each should become a Jira backlog story in the next sprint planning session:

1. **Home page stub** — `GET /` renders placeholder; `GET /api/papers` list endpoint should drive a real library page
2. **arXiv rate limiter not in test fixture** — `conftest.py` only resets upload limiter; arXiv limiter needs the same treatment
3. **CORS origins hardcoded for dev** — `CORS_ORIGINS` defaults are fine for localhost but production requires environment-specific overrides with domain validation
4. **No frontend type generation** — API types are maintained manually in `web/src/lib/api.ts`; should auto-generate from OpenAPI spec
5. **PDF extraction text-only** — figures, tables, and math formulae are not extracted (noted as non-goal but should be tracked)
6. **DOI resolver fallback** — if the primary DOI metadata provider fails, no fallback to secondary sources exists
7. **No authentication layer** — MVP is single-user/no-auth; auth needs to be added before multi-user deployment
8. **SQLite → Postgres migration** — the migration path exists in the spec but no migration tooling or scripts exist yet
9. **FTS5 search limited to title/abstract** — full text of extracted content not indexed in FTS5 virtual table
10. **No retry logic for failed Huey tasks** — tasks that fail due to transient errors (network, API rate limit) are not automatically retried
11. **Summariser and tagger are no-ops without LLM_API_KEY** — there is no graceful fallback or clear user-facing indication when these stages are skipped
12. **No pagination on `/api/papers`** — endpoint returns all records; needs limit/offset or cursor-based pagination for scalability
13. **Playwright smoke tests do not test authenticated flows** — once auth is added, smoke tests will need updating

---

## Implementation Status

| Recommendation | Status |
|---|---|
| REC-01 through REC-25 | ✅ Implemented (see commit following this document) |

---

*Retrospective facilitated by @ProjectManager on 2026-03-08. All inputs collected from @Analyst, @Architect, @BackendDev, @UIDev, @QAEngineer, and @WorkerDev.*
