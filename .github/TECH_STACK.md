# Tech Stack Manifest

> **Purpose:** Single source of truth for every technology binding used across `.github/` process files and agent instructions. When starting a new epic, @Architect reviews each slot and updates it to match the project's Architecture Specification. All downstream files inherit from this manifest.

---

## Stack Slots

| Slot | Current Value | Notes |
|------|--------------|-------|
| `BACKEND_FRAMEWORK` | FastAPI | Python async web framework |
| `VALIDATION_LIBRARY` | Pydantic v2 | Request/response schemas; `BaseSettings` for config |
| `ORM` | SQLAlchemy | With SQLModel as optional convenience layer |
| `DATABASE_TECH` | SQLite (MVP) | Future migration path to Postgres |
| `MIGRATION_TOOL` | Alembic | Schema versioning |
| `TASK_QUEUE` | Huey (SQLite-backed) | Async task processing; swap for Celery + Redis at scale |
| `PROCESSING_LIBRARY` | PyMuPDF (fitz) | PDF text extraction |
| `FRONTEND_FRAMEWORK` | Next.js 16 (App Router) | React Server Components by default |
| `CSS_FRAMEWORK` | TailwindCSS | Utility-first styling |
| `STATE_MANAGEMENT` | TanStack Query (React Query) | Server-state caching and polling |
| `TEST_BACKEND` | pytest + pytest-asyncio | Unit and integration tests |
| `TEST_FRONTEND` | Vitest | Component-level tests |
| `TEST_E2E` | Playwright | Headless browser testing |
| `API_PORT` | 8000 | FastAPI default; fallback 8001 |
| `FRONTEND_PORT` | 3000 | Next.js default; Playwright uses 3002 |
| `VENV_DIR` | `venv/` | Python virtual environment directory |

---

## Cross-Reference — Where Each Slot Is Used

Files marked **[concrete]** use the actual technology name in agent instructions (Group B).
Files marked **[indirect]** reference this manifest or the Architecture Spec instead of inlining tech names (Group A).

| Slot | Files |
|------|-------|
| `BACKEND_FRAMEWORK` | **[concrete]** `agents/backend-dev.agent.md`, `agents/qa-engineer.agent.md`, `agents/architext.agent.md`, `copilot-instructions.md` §3 · **[indirect]** `docs/OPERATING_PROCEDURES.md` §5.1, `skills/refinement-checklist/SKILL.md`, `skills/tep-review/SKILL.md` |
| `VALIDATION_LIBRARY` | **[concrete]** `agents/backend-dev.agent.md`, `copilot-instructions.md` §2, §3 · **[indirect]** `docs/OPERATING_PROCEDURES.md` §5.1 |
| `ORM` | **[concrete]** `agents/backend-dev.agent.md` · **[indirect]** `docs/OPERATING_PROCEDURES.md` §5.1 |
| `DATABASE_TECH` | **[concrete]** `agents/architext.agent.md`, `copilot-instructions.md` §3 · **[indirect]** `docs/OPERATING_PROCEDURES.md` §5.1, `skills/refinement-checklist/SKILL.md`, `skills/tep-review/SKILL.md` |
| `MIGRATION_TOOL` | **[concrete]** `agents/backend-dev.agent.md` |
| `TASK_QUEUE` | **[concrete]** `agents/worker-dev.agent.md`, `agents/backend-dev.agent.md`, `agents/architext.agent.md`, `copilot-instructions.md` §2 · **[indirect]** `docs/OPERATING_PROCEDURES.md` §5.1, `skills/refinement-checklist/SKILL.md`, `skills/tep-review/SKILL.md` |
| `PROCESSING_LIBRARY` | **[concrete]** `agents/worker-dev.agent.md` |
| `FRONTEND_FRAMEWORK` | **[concrete]** `agents/ui-dev.agent.md`, `agents/qa-engineer.agent.md`, `agents/architext.agent.md`, `copilot-instructions.md` §1, §3 · **[indirect]** `docs/OPERATING_PROCEDURES.md` §5.2, `skills/refinement-checklist/SKILL.md`, `skills/tep-review/SKILL.md` |
| `CSS_FRAMEWORK` | **[concrete]** `agents/ui-dev.agent.md` |
| `STATE_MANAGEMENT` | **[concrete]** `agents/ui-dev.agent.md` |
| `TEST_BACKEND` | **[concrete]** `agents/qa-engineer.agent.md`, `agents/backend-dev.agent.md` |
| `TEST_FRONTEND` | **[concrete]** `agents/ui-dev.agent.md` |
| `TEST_E2E` | **[concrete]** `agents/qa-engineer.agent.md`, `agents/ui-dev.agent.md`, `copilot-instructions.md` §3 · **[indirect]** `docs/OPERATING_PROCEDURES.md` §6.2 |
| `API_PORT` | **[concrete]** `agents/ui-dev.agent.md` · **[indirect]** `docs/OPERATING_PROCEDURES.md` §3.3 |
| `FRONTEND_PORT` | **[concrete]** `agents/ui-dev.agent.md` · **[indirect]** `docs/OPERATING_PROCEDURES.md` §3.3 |

---

## New Epic Checklist

When @Architect initialises a new epic, complete these steps to align the `.github/` framework:

1. **Create the Architecture Specification** from `.github/docs/templates/ARCHITECTURE_SPECIFICATION-TEMPLATE.md`.
2. **Review each Stack Slot** in the table above. Update `Current Value` and `Notes` to match the new project's spec.
3. **Update Group B (concrete) files** — edit each agent file and `copilot-instructions.md` to reflect the new technology names. The `<!-- Tech bindings from .github/TECH_STACK.md -->` comments in those files flag the lines to update.
4. **Group A (indirect) files** need no tech-name edits — they already point to the Architecture Spec or this manifest.
5. **Update `plans/TEP-TEMPLATE.md`** — refresh the example snippets in §1, §4, and §6 to use the new stack's idioms.
6. **Commit** all changes in a single `[EPIC-ID] Initialise tech stack` commit.
