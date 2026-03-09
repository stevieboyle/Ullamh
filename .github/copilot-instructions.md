# Repository Rules & Global Standards

## 1. Development Environment (Unix Remote)
- **Virtual Environment:** Use the Python virtual environment in `venv/` for all backend/worker tasks. The venv lives at `venv/` (not `.venv/`) in the project root.
- **Verification:** Always run `which python` or `python --version` before executing scripts to ensure the `venv` is active (`source venv/bin/activate`).
- **No Global Installs:** Never use `sudo pip`. All dependencies must be recorded in `requirements.txt` immediately after installation.
- **Node.js:** Use `npm` for frontend tasks; ensure `package-lock.json` is updated.

## 2. Project Ceremonies & Artifacts
- **The Pre-flight Check:** A mandatory huddle between @Analyst, @Architect, and the assigned Dev is required before moving **any** story to "In Progress" — this includes feature stories, spike stories, and test/enablement stories without exception.
- **Technical Execution Plan (TEP):**
  - Path: `.github/plans/TEP-[STORY-ID].md`.
  - Use the canonical template at `.github/plans/TEP-TEMPLATE.md`.
  - Content: Library versions, target file paths, model schemas, API signatures, **and a Cross-Cutting Concerns section** covering CORS, auth, rate limiting, logging, and error handling as applicable.
  - Approval: Must receive a "LGTM" or approval from @Architect before coding begins. @PM must verify a TEP link exists on the Jira ticket before approving the In Progress transition.
- **Spikes:** Every spike story must produce an Architecture Decision Record (ADR) saved to `docs/adr/ADR-[STORY-ID].md`. Use the template at `.github/docs/templates/ADR-TEMPLATE.md`. The decision, alternatives considered, and rationale must all be documented before the spike is closed.
- **Steel Thread Design:**
  - @Architect maintains `docs/design/STEEL_THREAD_DESIGN.md` (template: `.github/docs/templates/STEEL_THREAD_DESIGN-TEMPLATE.md`).
  - This maps the end-to-end flow through the backend, task queue, and database layers (see `.github/TECH_STACK.md`).
  - Devs must propose "Delta Updates" if implementation deviates from the spec. A Delta Update is required when a change affects **more than one file** or **any public API surface** (routes, schemas, task signatures).
- **Document Library:** Templates for all project documents (Architecture Spec, Dependency Plan, Startup Guide, Operations Runbook, User Guide, Pipeline Parameters, Retrospective, ADR, MVP Functionality, Architecture Guide, Steel Thread Design) are in `.github/docs/templates/`. Reference examples (fully-populated) are in `.github/docs/samples/`. See `.github/docs/OPERATING_PROCEDURES.md` §11 for the full creation-trigger table.

## 3. Implementation Standards
<!-- Tech bindings from .github/TECH_STACK.md — update when a slot changes -->
- **Backend:** See `.github/TECH_STACK.md` for the approved backend framework, validation library, ORM, and database.
- **Frontend:** See `.github/TECH_STACK.md` for the approved frontend framework. Use Server Components by default; Client Components only for leaf nodes.
- **Async Logic:** Tasks must be idempotent. Always check for existing records before processing.
- **Testing:** No story is "Done" until @QAEngineer runs tests in the Unix terminal and reports a pass.
- **Test File Naming:** Backend test files must be named `test_<feature>.py` under `tests/`. Frontend/component test files must be named `<Component>.test.tsx`. E2E test files must be named `<scenario>.spec.ts` under `web/tests/` only — do not place `.spec.ts` files elsewhere.

## 3a. Definition of Done (mandatory gate for all stories)
A story is **not Done** until all of the following are confirmed:
- [ ] All acceptance criteria checkboxes are checked
- [ ] @QAEngineer has run the full test suite and reported a pass
- [ ] A TEP exists at `.github/plans/TEP-[STORY-ID].md` and was approved before coding began
- [ ] Any new dependencies are recorded in `requirements.txt` (backend) or `package.json` (frontend)
- [ ] Any public API changes (routes, schemas, task signatures) are reflected in the Architecture Spec (Delta Update proposed to @Architect)
- [ ] Any documentation artifacts triggered by this story (see `.github/docs/OPERATING_PROCEDURES.md` §11) are created or updated from their templates in `.github/docs/templates/`
- [ ] Any deferred items or tech debt incurred are captured as follow-on Jira tickets by @PM
- [ ] The Jira ticket status has been updated to Done by @PM

## 4. Communication & Jira Sync
- **Jira Status:** @ProjectManager must update the Jira ticket status via MCP as the story moves through the TEP, Coding, and QA phases.
- **Handoffs:** Use the `team-handoff` skill for every inter-agent transition.