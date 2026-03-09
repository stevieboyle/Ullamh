# Reference Story: [EPIC_ID]-[STORY-NUMBER]
## Title: [WORKER] Implement [CORE_DOMAIN_ENTITY] Processing Service & Job Initialization

> **This is a generic template.** Replace all `[PLACEHOLDER]` values with your project's domain entities, framework names, and spec section references before use. See `.github/docs/samples/REFERENCE_STORY-SAMPLE.md` for a fully-populated example.

---

### 1. User Narrative
> **As a** [USER_PERSONA],
> **I want** the system to automatically process [CORE_DOMAIN_ENTITY] and extract key data immediately after it is submitted,
> **so that** I don't have to manually enter details and can see the processing status in real-time.

---

### 2. Technical Context (The "Agent's Map")
* **Target Layer:** [TASK_QUEUE] Worker / Backend Service.
* **Relevant Spec:** `[EPIC_ID]-architecture-specification.md`.
* **Primary Sections:** Section 5 (Processing Pipeline), Section [N] (Security), Section [N] (Open Decisions).
* **Key Files to Create/Modify:**
    * `[WORKER_DIR]/tasks.py` (New Task definition)
    * `[SERVICES_DIR]/[domain]_processor.py` (Core Processing Logic)
    * `src/db/models.py` (Update [CORE_DOMAIN_ENTITY] status enum)

---

### 3. Acceptance Criteria (AC)
- [ ] **Task Trigger:** A [TASK_QUEUE] task is successfully dispatched when a [CORE_DOMAIN_ENTITY] record is created in [DATABASE_TECH] with status `PENDING`.
- [ ] **Processing Engine:** Use `[PROCESSING_LIBRARY]` for [processing approach] as per the decision in Section [N] of the spec.
- [ ] **Fallback Behaviour:** If [CORE_DOMAIN_ENTITY] data is incomplete, [describe fallback strategy — e.g., extract a snippet for heuristic resolution].
- [ ] **Idempotency:** Re-running the task on the same [ENTITY_ID] must overwrite previous results without creating duplicate `[ENTITY]` records.
- [ ] **Status Updates:** The task must update the [DATABASE_TECH] `[ENTITY]` record to `PROCESSING` at start and `[INTERMEDIATE_STATUS]` upon successful completion.

---

### 4. Definition of Done (DoD) for Agents
* **Type Safety:** No `any` types in TypeScript or untyped variables in Python (use [VALIDATION_LIBRARY]).
* **Testing:** A unit test in `tests/[layer]/test_[domain]_processor.py` confirms successful processing of a representative sample [CORE_DOMAIN_ENTITY].
* **Performance:** Processing of a [typical size] [CORE_DOMAIN_ENTITY] must complete in < [N] seconds on the worker.
* **Documentation:** Add the new service method signature to the relevant internal docs.

---

### 5. Edge Cases & Constraints
* **[Error Condition 1]:** Return a specific `[FAILED_REASON]` status rather than throwing an unhandled exception. (HTTP `[status_code]` if applicable)
* **[Error Condition 2]:** If [condition], flag the record as `[DEFERRED_STATUS]` ([reason — e.g., out of scope for MVP]).
* **[Size/Rate Constraint]:** Log a warning and skip processing if the [CORE_DOMAIN_ENTITY] exceeds `[LIMIT]`.

---

### 6. Alignment with [EPIC_ID] Spec
* **Security:** Ensure the worker accesses [CORE_DOMAIN_ENTITY] from storage using the access pattern defined in Section [N] of the spec.
* **Scaling:** Ensure the database connection is closed properly at the end of the [TASK_QUEUE] task to prevent connection pooling exhaustion.

---

### 7. Cross-Cutting Concerns
* **CORS/Auth impact:** [State how this story affects auth/CORS, or state "N/A — worker task, no HTTP surface."]
* **Rate-limiting impact:** [State if any rate-limited endpoints are called, or "N/A."]
* **Logging/Observability:** [Log points: task received, processing start, processing complete, each failure branch.]
* **Error propagation:** [How failures surface to the job status model — which enum value is set on error.]