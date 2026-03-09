# Reference Story Sample: KAN-4-101
## Title: [WORKER] Implement PDF Text Extraction Service & Job Metadata Initialization

> **This is a sample document.** It contains project-specific content from the **Meitheal (KAN-4)** project and is provided as a concrete reference showing the level of detail expected in a refined backend/worker story. Do **not** use this as a starting point for new stories. Use the generic template at `.github/skills/refinement-checklist/assets/reference-story.md` instead.

---

### 1. User Narrative
> **As a** Researcher,
> **I want** the system to automatically extract clean text and basic metadata (title, authors) immediately after a PDF is uploaded,
> **so that** I don't have to manually enter paper details and can see the processing status in real-time.

---

### 2. Technical Context (The "Agent's Map")
* **Target Layer:** Celery Worker / Backend Service.
* **Relevant Spec:** `KAN-4-architecture-specification.md`.
* **Primary Sections:** Section 5 (Processing Pipeline), Section 11 (Security), Section 15 (Open Decisions).
* **Key Files to Create/Modify:**
    * `src/workers/tasks.py` (New Task definition)
    * `src/services/pdf_extractor.py` (Core Extraction Logic)
    * `src/db/models.py` (Update Paper status enum)

---

### 3. Acceptance Criteria (AC)
- [ ] **Task Trigger:** A Celery task is successfully dispatched when a paper record is created in Postgres with status `PENDING`.
- [ ] **Extraction Engine:** Use `PyMuPDF` (fitz) for text extraction as per the "text-first" decision in Section 15.
- [ ] **Metadata Fallback:** If PDF internal metadata is empty, extract the first 1000 characters of text to be used by the `@Analyst` for title heuristic resolution.
- [ ] **Idempotency:** Re-running the task on the same File ID must overwrite previous extraction results without creating duplicate `Paper` records.
- [ ] **Status Updates:** The task must update the Postgres `Paper` record to `PROCESSING` at start and `METADATA_PENDING` upon successful completion.

---

### 4. Definition of Done (DoD) for Agents
* **Type Safety:** No `any` types in TypeScript or `Untyped` variables in Python (use Pydantic v2).
* **Testing:** A unit test in `tests/workers/test_pdf_extractor.py` confirms successful text retrieval from a sample double-column PDF.
* **Performance:** Extraction of a 20-page PDF must complete in < 5 seconds on the worker.
* **Documentation:** Add the new service method signature to `docs/internal/api-services.md`.

---

### 5. Edge Cases & Constraints
* **Encrypted PDFs:** Return a specific `FAILED_ENCRYPTED` status rather than throwing an unhandled exception.
* **Image-Only PDFs:** If extracted text length is < 200 characters, flag the record as `NEEDS_OCR` (OCR is out of scope for MVP).
* **Large Files:** Log a warning and skip processing if the PDF exceeds 50MB.

---

### 6. Alignment with KAN-4 Spec
* **Security:** Ensure the worker reads the PDF from the private object storage using the signed URL logic defined in Section 11.
* **Scaling:** Ensure the database connection is closed properly at the end of the Celery task to prevent connection pooling exhaustion.

---

### 7. Cross-Cutting Concerns
* **CORS/Auth impact:** N/A — this is a worker task with no HTTP surface.
* **Rate-limiting impact:** N/A — task is triggered internally, not by an external rate-limited call.
* **Logging/Observability:** Log at task receipt, extraction start, extraction complete, and on each failure branch (encrypted, image-only, oversized).
* **Error propagation:** On any unhandled exception, set `Paper.status = FAILED` and `Job.status = FAILED`. The API's job-status polling endpoint surfaces this to the frontend.
