# Operating Procedures — [PROJECT_NAME]

**Document ID:** OP-001
**Version:** 1.0
**Effective Date:** 2026-03-07
**Owner:** @ProjectManager
**Status:** Active

---

## Table of Contents

1. [Purpose & Scope](#1-purpose--scope)
2. [Standard Story Workflow](#2-standard-story-workflow)
3. [Pre-flight Checklist](#3-pre-flight-checklist)
4. [TEP Standard](#4-tep-standard)
5. [Implementation Standards](#5-implementation-standards)
6. [QA Validation Standard](#6-qa-validation-standard)
7. [Handoff Protocol](#7-handoff-protocol)
8. [Jira Workflow](#8-jira-workflow)
9. [Living Spec Policy](#9-living-spec-policy)
10. [Success Patterns](#10-success-patterns)

---

## 1. Purpose & Scope

This document governs the end-to-end execution process for all stories under the [PROJECT_NAME] epic (**[EPIC_ID]**) and any future epics in this repository. It codifies the procedures, standards, and handoff protocols that every participating agent — @Analyst, @Architect, @BackendDev, @WorkerDev, @UIDev, @QAEngineer, and @ProjectManager — must follow to take a Jira ticket from **Backlog** to **Done**.

### What This Document Covers

- The mandatory sequence of gates every story passes through
- The content requirements for Technical Execution Plans (TEPs)
- Implementation constraints applicable to all code produced in this repository
- QA validation requirements before any ticket can be closed
- The handoff protocol that governs inter-agent transitions
- The Jira status lifecycle and ownership of each transition
- The process for proposing and approving changes to the canonical architecture spec

### What This Document Does Not Cover

- Business requirements (owned by @Analyst in Jira)
- Deployment and infrastructure beyond local development (covered in a future OP-002)
- Security incident response

### Authority

This document supersedes all informal conventions. In the event of conflict between this SOP and a TEP, this SOP takes precedence unless a formal Delta Update has been approved by @Architect and recorded in the architecture spec.

All agents must strictly adhere to .github/docs/OPERATING_PROCEDURES.md. No story may move to 'In Progress' without a verified TEP in .github/plans/, and no story is 'Done' without a documented Two-Phase QA pass

---

## 2. Standard Story Workflow

Every story follows this exact sequence. No step may be skipped. The responsible agent for each gate is listed explicitly.

```
Backlog → Next Up → Pre-flight → TEP Review → In Progress → Validation → Done
```

| Step | Gate Name | Owner | Entry Condition | Exit Condition |
|------|-----------|-------|-----------------|----------------|
| 1 | **Backlog Refinement** | @Analyst | Story exists in Jira | Acceptance criteria written; story points estimated |
| 2 | **Dependency Mapping** | @Architect | Story refined | All "Blocks" links added to Jira; steel thread updated if affected |
| 3 | **Ticket to Next Up** | @ProjectManager | Dependencies mapped | Ticket moved to "Next Up"; all blocking tickets resolved or in flight |
| 4 | **Pre-flight Check** | @Analyst + @Architect + Assigned Dev | Ticket is "Next Up" | All three agents have signed off on the Pre-flight Checklist (Section 3) |
| 5 | **TEP Authored** | Assigned Dev | Pre-flight complete | TEP file committed at `.github/plans/TEP-[STORY-ID].md` |
| 6 | **TEP Review & Approval** | @Architect | TEP committed | @Architect posts "LGTM" comment on the TEP PR or Jira ticket |
| 7 | **Ticket to In Progress** | @ProjectManager | TEP approved | Ticket moved to "In Progress"; start date recorded |
| 8 | **Implementation** | Assigned Dev | Ticket is "In Progress" | All TEP DoD checklist items checked; handoff package drafted |
| 9 | **Handoff Posted** | Assigned Dev | Implementation complete | Handoff package posted as Jira comment (Section 7) |
| 10 | **QA Routing** | @ProjectManager | Handoff received | Handoff forwarded to @QAEngineer; ticket held at "In Progress" |
| 11 | **QA Validation** | @QAEngineer | Handoff received | Both Phase 1 and Phase 2 QA checks pass (Section 6) |
| 12 | **Ticket to Validation** | @ProjectManager | QA pass confirmed | Ticket moved to "Validation"; QA report posted as Jira comment |
| 13 | **Review** | @Architect | Ticket in "Validation" | @Architect confirms implementation matches TEP and architecture spec |
| 14 | **Ticket to Done** | @ProjectManager | Review approved | Ticket moved to "Done"; DoD comment posted |

### Failure Paths

- **Pre-flight fails:** Ticket stays in "Next Up". @ProjectManager records the blocking condition as a Jira comment and assigns resolution to the appropriate agent.
- **TEP rejected:** @Architect posts specific required changes. Dev revises and resubmits. Ticket does not advance to "In Progress" until approval is granted.
- **QA fails:** Ticket stays in "In Progress". @ProjectManager posts failure details as a Jira comment and routes findings back to the originating dev. @QAEngineer documents exact reproduction steps.
- **Review fails:** Ticket returns to "In Progress" with @Architect's comments. A Delta Update may be required before re-review.

---

## 3. Pre-flight Checklist

The Pre-flight Check is a mandatory synchronous gate. @Analyst, @Architect, and the assigned Dev must each verify their respective sections before any ticket moves to "In Progress". Completion is recorded as a Jira comment.

### 3.1 @Analyst Checklist

- [ ] Acceptance criteria are unambiguous and testable
- [ ] All upstream dependencies (blocking tickets) are resolved or have a confirmed owner
- [ ] The story scope has not drifted since refinement; if it has, the ticket has been updated
- [ ] Any external API contracts or data schemas this story depends on are documented in Jira

### 3.2 @Architect Checklist

- [ ] The story's implementation path is consistent with `docs/design/STEEL_THREAD_DESIGN.md`
- [ ] The Dependency DAG in Jira is up to date (all "Blocks" links verified)
- [ ] No architectural decision required by this story has been left open
- [ ] If this story touches the steel thread, the impact on downstream stories has been assessed
- [ ] The TEP template (`.github/plans/TEP-TEMPLATE.md`) has been sent to or confirmed available to the assigned Dev

### 3.3 Assigned Dev Checklist

**Environment Prerequisites (mandatory — failure here caused the Node.js incident in Layer 0)**

- [ ] Python version verified: `python --version` returns the required version; `venv/` is present or can be created
- [ ] Node.js version verified: `node --version` returns the required version; if absent, install via `nvm` before proceeding
- [ ] `nvm` is available; if the story requires a specific Node version, run `nvm use <version>` and verify
- [ ] All required system-level tools (`git`, `curl`, `jq`, etc.) are present
- [ ] Port availability verified: default ports (see `.github/TECH_STACK.md` for `API_PORT` and `FRONTEND_PORT`) are free; if not, the TEP must document the fallback port and CI must be notified
- [ ] Virtual environment activated: `which python` resolves to `venv/bin/python`
- [ ] `requirements.txt` (backend) or `package.json` (frontend) dependencies install cleanly in the target environment

**Story Readiness**

- [ ] The acceptance criteria are understood; any ambiguity has been escalated to @Analyst
- [ ] The TEP template has been reviewed; no required section is unclear
- [ ] Estimated effort is still accurate given the current environment state

**Pre-flight Sign-off Comment Format**

Each agent posts to the Jira ticket:

```
PRE-FLIGHT SIGN-OFF — [STORY-ID]
Agent: @[AgentName]
Date: YYYY-MM-DD
Status: APPROVED / BLOCKED
Blocking condition (if any): [description]
```

---

## 4. TEP Standard

Every Technical Execution Plan is a binding contract between the Dev, @Architect, and @QAEngineer. It must be committed to `.github/plans/TEP-[STORY-ID].md` before any code is written.

The first backend and frontend scaffold stories are the canonical reference implementations. When in doubt about the level of detail required, match those TEPs.

### 4.1 Required Sections

Every TEP must contain all eight sections below. A TEP missing any section will be rejected by @Architect.

---

#### Section 1: Library Versions Table

A pinned dependency table with justification for each version choice. "Latest" is not an acceptable version specifier.

| Library | Pinned Version | Justification |
|---------|---------------|---------------|
| `[framework]` | `x.y.z` | [Purpose / compatibility notes] |
| `[library]` | `x.y.z` | [Purpose / compatibility notes] |

**CLI Command Verification (mandatory — failure here caused incidents in previous projects):**
For every CLI command referenced in the TEP, verify the command exists in the installed version before including it. Document the verification step inline:

```
Verified: `[command]` — available in [framework] [version]. (Verified YYYY-MM-DD)
```

---

#### Section 2: Target File Paths & Directory Tree

The exact filesystem structure the story will produce. Every new file and modified file must be listed.

```
src/
├── __init__.py          # NEW
├── main.py              # NEW — Application entry point
└── config.py            # NEW — Settings with env-var validation
```

---

#### Section 3: Key Interfaces / Schemas

All validation models (backend) and TypeScript interfaces (frontend) introduced or modified by this story. Include field names, types, validators, and defaults.

```python
class Settings(BaseSettings):
    database_url: str = "sqlite:///./app.db"
    model_config = SettingsConfigDict(env_file=".env", env_file_encoding="utf-8")
```

```typescript
interface JobStatus {
  jobId: string;
  status: "pending" | "running" | "complete" | "failed";
}
```

---

#### Section 4: API Signatures

Every HTTP endpoint introduced or modified. Include method, path, request body schema, response schema, and status codes.

```
GET /health
  Response 200: { "status": "ok", "version": "0.1.0" }

POST /jobs
  Request body: { "item_id": string }
  Response 201: { "job_id": string, "status": "pending" }
  Response 409: { "detail": "Job already exists for this item" }
```

---

#### Section 5: Environment Variables

Every environment variable the story introduces or depends on.

| Variable | Default | Required | Description |
|----------|---------|----------|-------------|
| `DATABASE_URL` | `sqlite+aiosqlite:///./data/app.db` | No | Async database connection string |

For each variable that can be overridden in CI or testing, document the override mechanism and verify it works (as per the `DATABASE_URL` lesson from Layer 0).

---

#### Section 6: Definition of Done Checklist

A binary checklist. All items must be checked before the Dev posts the handoff package.

- [ ] All target files exist at the documented paths
- [ ] Application starts without errors (`uvicorn` / `next dev`)
- [ ] All listed API endpoints return expected status codes (verified via curl)
- [ ] CLI commands verified against installed version
- [ ] Environment variable overrides verified
- [ ] Port fallback documented if default port was unavailable
- [ ] No linting errors
- [ ] `requirements.txt` / `package.json` updated with all new dependencies

---

#### Section 7: Spec Alignment

Explicit statement of which version of `docs/design/STEEL_THREAD_DESIGN.md` this TEP implements and any known deviations.

```
Implements: Architecture Spec v0.2 (committed 2026-02-15)
Deviations: None
```

---

#### Section 8: Constraints

Hard constraints the Dev must not violate. Copy the relevant constraints from Section 5 of this SOP and add any story-specific constraints.

---

### 4.2 TEP Review Process

1. Dev commits the TEP to `.github/plans/TEP-[STORY-ID].md` and posts the path in the Jira ticket comment.
2. @Architect reviews within one working session.
3. @Architect either posts "LGTM — TEP approved for KAN-X" or lists required changes with specific section references.
4. Dev addresses all required changes and requests re-review.
5. No code is written until @Architect's approval comment is posted.

---

## 5. Implementation Standards

These constraints apply to all code produced in this repository regardless of story or agent.

### 5.1 Backend (Python)

- **Framework:** See `.github/TECH_STACK.md` for the approved backend framework, validation library, ORM, and database. Do not use Pydantic v1 syntax (`class Config:` is deprecated; use `model_config = SettingsConfigDict(...)`).
- **Database:** See `TECH_STACK.md` — `ORM` slot and `DATABASE_TECH` slot. All models must be written for future migration to a production database.
- **Virtual Environment:** Always located at `venv/` relative to the repository root. Activate with `source venv/bin/activate` before any Python command. The `copilot-instructions.md` documents this path — do not deviate.
- **Dependencies:** All new dependencies must be added to `requirements.txt` immediately upon installation with the exact pinned version.
- **No Global Installs:** `sudo pip` is prohibited.
- **Idempotency:** All async tasks and database write operations must be idempotent. Check for existing records by a unique key before inserting.
- **Settings:** Use the validation library’s settings class with `env_file=".env"`. Never hardcode secrets or connection strings.
- **Async:** Use `async def` for all route handlers. Use `await` for all database operations.

### 5.2 Frontend

- **Framework:** See `.github/TECH_STACK.md` — `FRONTEND_FRAMEWORK` slot.
- **Component Default:** All components are React Server Components by default. Add `"use client"` only to leaf-node components that require browser APIs or React hooks. Justify every directive in a comment.
- **Node.js Management:** Use `nvm` to install and switch Node.js versions. The required version is documented in `.nvmrc` at the repository root.
- **CLI Commands:** Verify every CLI command against the installed framework version. Document the verification step inline in the TEP.
- **Dependencies:** All new packages must be committed with an updated `package-lock.json`.
- **TypeScript:** Strict mode enabled. No `any` types without explicit justification.
- **Ports:** Default is 3000. Fallback is 3001. Document the fallback in the TEP.

### 5.3 Cross-Cutting

- **No secrets in source control.** Use `.env` files (git-ignored). Document all required env vars in `.env.example`.
- **No force pushes** to `main` or any branch with an open PR.
- **Commit messages:** `[STORY-ID] <imperative verb> <object>` (e.g., `[STORY-1] Add health endpoint`).
- **File paths** must match the Target File Paths section of the TEP exactly.

---

## 6. QA Validation Standard

QA is a two-phase process. Both phases must pass before @QAEngineer posts a QA pass report.

### 6.1 Phase 1 — Static & Curl Checks

**Backend checks:**
- [ ] Server starts without errors; no import errors in stdout
- [ ] `GET /health` returns `{"status": "ok"}` with HTTP 200
- [ ] `GET /docs` returns Swagger UI HTML (HTTP 200)
- [ ] All TEP API Signatures return documented status codes
- [ ] `DATABASE_URL` override verified (not cached from a previous start)

**Frontend checks:**
- [ ] `npm run dev` starts on the expected port (document the port in the TEP; default 3000, fallback 3001)
- [ ] `curl http://localhost:3001` returns non-empty HTML with HTTP 200 (substitute the TEP-documented port)
- [ ] `npx tsc --noEmit` — 0 errors
- [ ] `npx eslint .` — 0 errors

### 6.2 Phase 2 — Browser Tests

Headless Chromium via the E2E test framework (see `.github/TECH_STACK.md` — `TEST_E2E` slot). Curl checks are insufficient to confirm working UI behaviour.

**Minimum required coverage:**

| Scenario | Assertion |
|----------|-----------|
| Frontend homepage loads | Title present; expected heading visible |
| Backend API docs loads | API documentation page renders and lists endpoints |
| Health endpoint via browser | Navigating to `/health` returns JSON with `status: "ok"` |
| No JS runtime errors | `page.on('pageerror')` listener reports no uncaught exceptions during page load |

**Test file location:** `web/tests/smoke.spec.ts`

**Pass criteria:** All tests green. Zero failures. Zero skipped tests.

**QA Pass Report format** (posted as Jira comment):

```
QA PASS REPORT — [STORY-ID]
Agent: @QAEngineer
Date: YYYY-MM-DD

Phase 1 — Static & Curl Checks: PASS
  - Health endpoint: 200 OK
  - Swagger UI accessible: YES
  - DATABASE_URL override verified: YES
  - Frontend build: clean

Phase 2 — Playwright Browser Tests: PASS
  - Tests run: N
  - Tests passed: N
  - Tests failed: 0
  - Console errors: None

Recommendation: APPROVE for transition to Validation
```

**QA Fail Report format:**

```
QA FAIL REPORT — [STORY-ID]
Agent: @QAEngineer
Date: YYYY-MM-DD

Failing checks:
  1. [Phase][Check name]: [exact error]
     Reproduction steps: [exact commands]
     Expected: [expected result]
     Actual: [actual result]

Recommendation: RETURN to [AssignedDev]
```

---

## 7. Handoff Protocol

Every agent-to-agent transition requires a formal handoff package posted as a Jira comment. No transition may be acknowledged without a complete package.

### 7.1 The Four-Field Package

```
HANDOFF PACKAGE — [STORY-ID]
From: @[AgentName]
To: @[AgentName]
Date: YYYY-MM-DD

## 1. Status
[What was completed. Reference TEP DoD checklist items. If any item is incomplete, explain why.]

## 2. Artifacts
[Every file created, modified, or deleted — full paths relative to the repo root.]

- Created: `src/main.py`
- Modified: `requirements.txt`

## 3. Context for Next
[How to start the service, known quirks, environment setup, port numbers, non-obvious decisions.]

## 4. Blockers
[Unresolved issues or known limitations. If none, write "None".]
```

### 7.2 Handoff Routing

| From | To | Condition |
|------|----|-----------|
| Assigned Dev | @QAEngineer (via @ProjectManager) | Implementation complete; DoD checked |
| @QAEngineer | @ProjectManager | QA result determined |
| @ProjectManager | Assigned Dev | QA fail; findings routed back |
| @ProjectManager | @Architect | QA pass; ticket in Validation |
| @Architect | @ProjectManager | Review complete |

### 7.3 Rejection Protocol

If @ProjectManager receives a package missing any field:
1. Post a Jira comment citing the missing fields.
2. Return ticket to the sending agent.
3. Do not route to @QAEngineer until a complete package is provided.

---

## 8. Jira Workflow

### 8.1 Status Flow

```
Backlog → Next Up → In Progress → Validation → Done
```

Backward movements (e.g., Validation → In Progress on review failure) are permitted only when directed by @ProjectManager.

### 8.2 Transition Ownership

| Transition | Owner | Trigger |
|------------|-------|---------|
| Backlog → Next Up | @ProjectManager | Pre-flight conditions met |
| Next Up → In Progress | @ProjectManager | TEP approved by @Architect |
| In Progress → Validation | @ProjectManager | QA pass report received |
| Validation → Done | @ProjectManager | @Architect review approved |
| Any → In Progress (regression) | @ProjectManager | QA fail or review rejection |

### 8.3 Label Standards

| Label | Applied When |
|-------|-------------|
| `pre-flight-approved` | Pre-flight sign-off complete |
| `tep-approved` | @Architect has posted LGTM |
| `qa-pass` | QA pass report posted |
| `qa-fail` | QA fail report posted (removed when resolved) |
| `delta-update-required` | Implementation deviates from architecture spec |
| `blocked` | External dependency unresolved |

### 8.4 Dependency Links

- All "Blocks" relationships established before a story moves from Backlog to Next Up.
- @Architect is responsible for the completeness of the Dependency DAG at the start of each layer.

---

## 9. Living Spec Policy

### 9.1 The Canonical Architecture Spec

`docs/design/STEEL_THREAD_DESIGN.md` is the single source of truth for the overall system architecture.

**Owner:** @Architect  
**Update cadence:** Updated at the end of each layer retrospective and whenever a Delta Update is approved.

### 9.2 Version Control

The canonical versioned document is `docs/[EPIC-ID]-architecture-specification.md` (e.g., v0.2). `docs/design/STEEL_THREAD_DESIGN.md` is the living design document — @Architect updates its header tag to match the spec version after each change. Every TEP's Spec Alignment section must reference the architecture specification version number it implements.

> **New project:** When starting a new epic, create the architecture specification from `.github/docs/templates/ARCHITECTURE_SPECIFICATION-TEMPLATE.md` and the steel thread design from `.github/docs/templates/STEEL_THREAD_DESIGN-TEMPLATE.md`. These documents must exist before any story is moved to "Next Up".

### 9.3 Delta Update Process

**When required:** The Dev cannot follow the spec as written due to a tool, library, or constraint discovered during implementation.

**Process:**
1. Dev stops implementing the deviating component.
2. Dev posts a Delta Update proposal as a Jira comment:

```
DELTA UPDATE PROPOSAL — [STORY-ID]
Author: @[DevName]
Date: YYYY-MM-DD

Spec reference: Architecture Spec v0.2, Section [N]
Proposed deviation: [What the implementation will do instead]
Rationale: [Why the spec cannot be followed as written]
Impact on downstream stories: [Affected tickets, or "None"]
```

3. @Architect reviews and approves or rejects.
4. If approved: @Architect updates `STEEL_THREAD_DESIGN.md`, increments the version, and posts the new version on the ticket. The TEP Spec Alignment section is updated.
5. If rejected: Dev must conform to the spec or escalate to @ProjectManager.

**No deviation is approved until @Architect has explicitly posted approval and updated the spec.**

### 9.4 Retrospective Updates

At the end of each layer, @Architect conducts a spec review, confirms accumulated Delta Updates are reflected, and increments the minor version (e.g., v0.2 → v0.3).

---

## 10. Success Patterns

These named patterns are extracted from Layer 0 execution. Cite them by name in TEPs, handoff packages, and retrospectives.

---

### `TEP-Gold-Standard`

**Origin:** First backend and frontend scaffold stories  
**Description:** A TEP that contains all eight required sections with sufficient detail that the assigned Dev can execute without asking any clarifying questions.  
**Key principle:** Ambiguity in a TEP is a defect, not a known unknown.

---

### `Two-Phase QA`

**Origin:** First backend and frontend scaffold stories  
**Description:** QA is never satisfied by static checks or curl responses alone. Phase 1 (curl/terminal) establishes baseline health. Phase 2 (Playwright browser tests) confirms real browser behaviour and JS execution.  
**Key principle:** A service that responds to curl may still fail in a browser due to JS errors or missing assets.

---

### `Playwright-Smoke-Gate`

**Origin:** First frontend QA pass  
**Description:** A minimum Playwright smoke test suite at `web/tests/smoke.spec.ts` that verifies the homepage loads, the Swagger UI is accessible, and no JS runtime errors occur. Runs as part of every QA validation pass.  
**Key principle:** Smoke tests are the floor, not the ceiling.

---

### `Dependency-DAG-First`

**Origin:** Layer 0 planning  
**Description:** Before any story in a layer moves to "Next Up", the full Dependency DAG for the layer is established in Jira using "Blocks" links. The steel thread is documented explicitly.  
**Key principle:** Discovering a dependency mid-execution is a process failure, not a normal occurrence.

---

### `Pre-flight-Environment-Verify`

**Origin:** Layer 0 — Node.js not pre-installed discovered mid-execution  
**Description:** Before a ticket moves to "In Progress", the assigned Dev must verify every runtime and toolchain dependency is present: Python version, `venv/` state, Node.js via `nvm`, required ports, and system tools.  
**Key principle:** "It should be installed" is not verification. Run the command and read the output.

---

### `Idempotent-Task-Design`

**Origin:** Architecture Spec v0.2  
**Description:** Every async task and database write operation checks for an existing record by a unique key before inserting. Duplicate submissions return the existing record.  
**Key principle:** A task that cannot be safely retried is a liability in an async system.

---

### `Checkpoint-TEP`

**Origin:** Layer 0 — session interruptions required state recovery  
**Description:** The TEP DoD Checklist doubles as a checkpoint log. As each item is completed, the Dev posts a Jira comment marking it done. Interrupted executions resume from the last confirmed checkpoint.  
**Key principle:** Progress must be externally observable and recoverable without the original agent's memory.

---

### `Four-Field-Handoff`

**Origin:** Layer 0 inter-agent transitions  
**Description:** Every handoff includes exactly four fields: Status, Artifacts, Context for Next, Blockers. Packages missing any field are rejected and returned.  
**Key principle:** The receiving agent should be able to continue without a synchronous conversation with the sending agent.

---

## 11. Document Library

Every epic in this repository maintains a structured document library in `docs/`. Templates for all document types are in `.github/docs/templates/`. Reference examples (fully-populated) are in `.github/docs/samples/`.

### 11.1 Document Creation Triggers

The following table defines when each document must be created, who creates it, and which template to use. A document is a **required artifact** if its trigger event has occurred — it must be created (not just planned) before the associated gate can close.

| Document | Template | Owner | Create Trigger | Update Trigger |
|---|---|---|---|---|
| `docs/[EPIC]-architecture-specification.md` | `ARCHITECTURE_SPECIFICATION-TEMPLATE.md` | @Architect | Epic kick-off — before any refinement begins | End-of-layer retrospective (minor version) |
| `docs/[EPIC]-dependency-plan.md` | `DEPENDENCY_PLAN-TEMPLATE.md` | @Architect | After spec is approved | When stories are added or priorities change |
| `docs/design/STEEL_THREAD_DESIGN.md` | `STEEL_THREAD_DESIGN-TEMPLATE.md` | @Architect | Before Layer 0 pre-flight | Each approved Delta Update |
| `docs/ARCHITECTURE_GUIDE.md` | `ARCHITECTURE_GUIDE-TEMPLATE.md` | @Architect | End of Layer 0 | End of each subsequent layer |
| `docs/STARTUP_GUIDE.md` | `STARTUP_GUIDE-TEMPLATE.md` | Scaffold Dev | End of Layer 0 (scaffold stories complete) | Any change to ports, commands, or env vars |
| `docs/OPERATIONS_RUNBOOK.md` | `OPERATIONS_RUNBOOK-TEMPLATE.md` | @PM + @BackendDev | Before first staging/production deployment | Deployment procedure changes; new incident response patterns |
| `docs/USER_GUIDE.md` | `USER_GUIDE-TEMPLATE.md` | @UIDev | End of MVP sprint | Any user-visible feature change |
| `docs/MVP-FUNCTIONALITY.md` | `MVP_FUNCTIONALITY-TEMPLATE.md` | @PM | After each sprint delivery or on MVP completion | Each subsequent release that changes delivered scope |
| `docs/pipeline-parameters.md` | `PIPELINE_PARAMETERS-TEMPLATE.md` | @WorkerDev | When first pipeline service (`src/services/`) is introduced | Any change to a prompt template, model id, threshold, or rate-limiter config |
| `docs/RETROSPECTIVE-[DATE].md` | `RETROSPECTIVE-TEMPLATE.md` | @PM | End of each sprint or at epic close | N/A (retrospectives are point-in-time; create a new file per retrospective) |
| `docs/adr/ADR-[STORY-ID].md` | `ADR-TEMPLATE.md` | Assigned Dev | On spike story closure — required before the spike ticket moves to Done | N/A (ADRs are immutable once approved) |

### 11.2 Document Ownership Responsibilities

| Agent | Owns | Action Required |
|---|---|---|
| @Architect | Architecture Specification, Dependency Plan, Steel Thread Design, Architecture Guide, ADRs | Create from template at trigger event; approve Delta Updates; increment spec version post-retro |
| @PM | Retrospectives, MVP Functionality, Operations Runbook (coordinate) | Create from template at trigger event; verify all required docs exist before epic gates close |
| Scaffold Dev / @BackendDev | Startup Guide | Create from template at end of Layer 0; update whenever setup instructions change |
| @WorkerDev | Pipeline Parameters | Create from template on first pipeline service; update with every parameter change |
| @UIDev | User Guide | Create from template at MVP; update with every user-visible feature change |

### 11.3 Gatekeeping Rule

@PM must verify that all documents whose trigger events have fired are created and non-skeleton before signing off on the associated Jira ticket. "Created from template" means the placeholders have been replaced — a file containing `[PROJECT_NAME]` is not complete.

### 11.4 Reference Examples

Four fully-populated sample documents are available in `.github/docs/samples/`:

| Sample | Maps to Template |
|---|---|
| `RETROSPECTIVE-SAMPLE.md` | `RETROSPECTIVE-TEMPLATE.md` |
| `OPERATIONS_RUNBOOK-SAMPLE.md` | `OPERATIONS_RUNBOOK-TEMPLATE.md` |
| `STARTUP_GUIDE-SAMPLE.md` | `STARTUP_GUIDE-TEMPLATE.md` |
| `STEEL_THREAD_DESIGN-SAMPLE.md` | `STEEL_THREAD_DESIGN-TEMPLATE.md` |

Consult these samples when creating the equivalent document for a new project to understand the expected level of detail.

---

## Appendix A: Quick Reference Card

```
Story Lifecycle:    Backlog → Next Up → In Progress → Validation → Done
Pre-flight owners:  @Analyst + @Architect + Assigned Dev
TEP approval:       @Architect ("LGTM" comment required)
QA phases:          Phase 1 (curl) + Phase 2 (Playwright) — both required
Handoff fields:     Status | Artifacts | Context for Next | Blockers
Spec location:      docs/design/STEEL_THREAD_DESIGN.md
TEP location:       .github/plans/TEP-[STORY-ID].md
Smoke tests:        web/tests/smoke.spec.ts
Python venv:        venv/ (source venv/bin/activate)
Node version mgr:   nvm (see .nvmrc)
Doc templates:      .github/docs/templates/  (11 templates)
Doc samples:        .github/docs/samples/    (reference examples)
TEP template:       .github/plans/TEP-TEMPLATE.md
```

---

## Appendix B: Document History

| Version | Date | Author | Summary |
|---------|------|--------|---------|
| 1.0 | 2026-03-07 | @ProjectManager | Initial release; codifies Layer 0 retrospective findings |
| 1.1 | 2026-03-08 | @ProjectManager | Add §11 Document Library; add template references throughout; Appendix A updated |
