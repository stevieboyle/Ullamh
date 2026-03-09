# TEP-KAN-14 — PDF Text Extraction & Basic Metadata

> **This is a sample document.** It contains project-specific content from the **Meitheal (KAN-4)** project and is provided as a concrete reference showing the level of detail expected in a Technical Execution Plan (TEP). Use the template at [`.github/plans/TEP-TEMPLATE.md`](../../plans/TEP-TEMPLATE.md) when creating TEPs for a new project.

**Story:** KAN-14  
**Author:** @Architect  
**Status:** ✅ LGTM — approved for implementation  
**Date:** 2026-03-07

---

## 1. Scope

Implement PDF text extraction and metadata harvesting inside the Huey worker.
Results are persisted to `PaperText` and may backfill `Paper.title`,
`Paper.abstract`, and `PaperAuthor` rows when those fields are currently `NULL`.

**Out of scope (separate stories):**
- Token-level chunking → KAN-15
- LLM summarization → KAN-16

---

## 2. Library Versions

| Library      | Version | Notes                                   |
|--------------|---------|-----------------------------------------|
| PyMuPDF      | 1.27.1  | `import fitz`; chosen in KAN-37 spike   |
| SQLAlchemy   | 2.0.41  | Sync session used inside Huey tasks     |
| Pydantic     | 2.11.1  | Schemas only — not used in this story   |
| FastAPI      | 0.115.12| No new endpoints required               |
| Huey         | 2.5.3   | Task decorator unchanged                |

`pymupdf==1.27.1` has been added to `requirements.txt`.

---

## 3. Target File Paths

| Action   | Path                            | Description                                        |
|----------|---------------------------------|----------------------------------------------------|
| **NEW**  | `src/services/pdf_extractor.py` | `PdfExtractor` service + `ExtractResult` dataclass |
| **MODIFY** | `src/worker/tasks.py`         | Integrate extraction into `process_paper()`        |

---

## 4. `PdfExtractor` Class API

### 4.1 `ExtractResult` dataclass

```python
# src/services/pdf_extractor.py
from dataclasses import dataclass, field

@dataclass
class ExtractResult:
    full_text: str                   # Concatenated text of all pages (may be empty string)
    title:     str | None            # From PDF /Title metadata, or None
    authors:   list[str]             # Parsed from PDF /Author metadata (comma-split), may be []
    abstract:  str | None            # From PDF /Subject metadata if present, else None
```

**Design notes:**
- `full_text` is never `None`; an empty string indicates a text-free (e.g. scanned image) PDF.
- `authors` is a `list[str]` to align with the `PaperAuthor` model's per-row design.
  Each element is one author name (stripped).
- `abstract` maps from the PDF `/Subject` metadata key, which some authoring tools
  (e.g. LaTeX `hyperref`) populate with the abstract. It is `None` when absent.

### 4.2 `PdfExtractor.extract()`

```python
# src/services/pdf_extractor.py
import fitz  # PyMuPDF

class PdfExtractor:
    """Extracts full text and basic metadata from a PDF byte payload."""

    def extract(self, pdf_bytes: bytes) -> ExtractResult:
        """Parse pdf_bytes and return an ExtractResult.

        Raises:
            fitz.FileDataError: if the bytes cannot be parsed as a valid PDF.
        """
        doc = fitz.open(stream=pdf_bytes, filetype="pdf")

        # ── Full text ─────────────────────────────────────────────────────────
        pages: list[str] = [page.get_text() for page in doc]
        full_text: str = "\n".join(pages)

        # ── Metadata ──────────────────────────────────────────────────────────
        meta: dict = doc.metadata or {}

        title: str | None = meta.get("title") or None

        raw_authors: str = meta.get("author") or ""
        authors: list[str] = [
            a.strip() for a in raw_authors.split(",") if a.strip()
        ]

        abstract: str | None = meta.get("subject") or None

        return ExtractResult(
            full_text=full_text,
            title=title,
            authors=authors,
            abstract=abstract,
        )
```

**Why `fitz.FileDataError` is not caught here:**  
Single-responsibility — the extractor raises, the task layer owns error-to-`Job`
mapping. This keeps `PdfExtractor` independently testable.

---

## 5. Modified `process_paper()` — Pseudocode

```python
# src/worker/tasks.py  (modified)

from src.db.models import (
    Job, JobStatus, JobStage,
    Paper, PaperAuthor, PaperFile, PaperText,
)
from src.services.pdf_extractor import PdfExtractor
from src.storage.local import LocalStorageBackend
from src.config import get_settings

@huey.task(retries=1, retry_delay=30)
def process_paper(paper_id: str, job_id: str) -> None:
    engine = _get_sync_engine()
    with Session(engine) as session:

        # ── Guard: job must exist ─────────────────────────────────────────────
        job = session.get(Job, job_id)
        if job is None:
            logger.error("Job %s not found — aborting", job_id)
            return

        # ── Idempotency: skip if already finished ─────────────────────────────
        if job.status == JobStatus.PROCESSED:
            logger.info("Job %s already PROCESSED — skipping", job_id)
            return

        job.status = JobStatus.RUNNING
        session.commit()

        try:
            # ── PARSE stage ───────────────────────────────────────────────────
            job.stage = JobStage.PARSE
            session.commit()

            # ── Load paper + its primary file ────────────────────────────────
            paper: Paper = session.get(Paper, paper_id)
            paper_file: PaperFile = paper.files[0]  # exactly one file at MVP

            settings = get_settings()
            storage = LocalStorageBackend(settings.storage_dir)
            pdf_bytes: bytes = storage.load(paper_file.storage_path)

            # ── Extract text & metadata ───────────────────────────────────────
            result = PdfExtractor().extract(pdf_bytes)
            # fitz.FileDataError bubbles up → caught by outer except block

            # ── Persist full text (upsert: handle retries gracefully) ─────────
            existing_text = session.get(PaperText, {"paper_id": paper_id})
            if existing_text:
                existing_text.full_text = result.full_text
            else:
                session.add(PaperText(paper_id=paper_id, full_text=result.full_text))

            # ── Backfill Paper metadata (only if currently NULL) ──────────────
            if paper.title is None and result.title:
                paper.title = result.title

            if paper.abstract is None and result.abstract:
                paper.abstract = result.abstract

            if not paper.authors and result.authors:
                for idx, author_name in enumerate(result.authors):
                    session.add(PaperAuthor(
                        paper_id=paper_id,
                        author_order=idx,
                        name=author_name,
                    ))

            # ── Mark complete ─────────────────────────────────────────────────
            job.status = JobStatus.PROCESSED
            session.commit()

        except Exception as e:
            logger.exception("Job %s (PARSE stage) failed: %s", job_id, e)
            job.status = JobStatus.FAILED
            job.error = str(e)
            session.commit()
            raise  # allow Huey's retry mechanism to fire (retries=1)
```

**Key behaviours:**
- `job.stage` is set to `PARSE` *before* the expensive I/O begins so the UI can
  display granular progress.
- `job.stage` is intentionally **not reset** after `PROCESSED` — it records which
  stage the job last executed, which is useful for debugging.
- The commit after `job.stage = PARSE` ensures the stage is visible to the API
  even if the task crashes hard (e.g. OOM).

---

## 6. DB Write Summary

| Model       | Operation | Condition                              |
|-------------|-----------|----------------------------------------|
| `PaperText` | INSERT or UPDATE `full_text` | Always (upsert for retry safety) |
| `Paper.title` | UPDATE | Only if `paper.title IS NULL` and `result.title` is not None |
| `Paper.abstract` | UPDATE | Only if `paper.abstract IS NULL` and `result.abstract` is not None |
| `PaperAuthor` | INSERT N rows | Only if `paper.authors` is empty and `result.authors` is non-empty |
| `Job.stage` | UPDATE → `PARSE` | At start of processing block |
| `Job.status` | UPDATE → `PROCESSED` or `FAILED` | On completion or exception |
| `Job.error` | UPDATE | Only on exception |

---

## 7. Edge Cases

| Scenario | Behaviour |
|----------|-----------|
| **Corrupt / unreadable PDF** | `fitz.open()` raises `fitz.FileDataError`; caught in outer `except Exception`, sets `job.status = FAILED`, `job.error = str(e)`. Huey retries once after 30 s; if still failing, job stays `FAILED`. |
| **Image-only (scanned) PDF** | `page.get_text()` returns `""` for each page. `full_text` is an empty string. This is **not** an error — `PaperText.full_text = ""` is valid. KAN-15 chunker should skip empty texts. |
| **PDF metadata title/authors missing** | `doc.metadata` values are empty strings `""`. The `or None` guard converts them to `None`, so `Paper` fields are not overwritten. |
| **PDF `/Author` with semicolons** | MVP splits on commas only. Semicolons produce a single author string. This is acceptable at MVP — KAN-33 (author normalisation) can improve this. |
| **`paper.files` is empty** | `IndexError` on `paper.files[0]`. Caught by `except Exception`, marks `FAILED`. This should never happen if the upload endpoint is correct. |
| **Retry: `PaperText` already exists** | `session.get(PaperText, ...)` detects the existing row and calls `UPDATE` instead of `INSERT`, preventing a `UNIQUE` constraint violation. |
| **Retry: `PaperAuthor` rows exist** | `if not paper.authors` guard prevents duplicate inserts. |

---

## 8. Testing Notes (@QAEngineer)

1. **Unit test `PdfExtractor`** — pass a real minimal PDF binary (or a PyMuPDF-generated
   one) to `extract()`; assert `full_text` is non-empty, `title` is captured.
2. **Unit test corrupt PDF** — pass `b"not a pdf"`, assert `fitz.FileDataError` raised.
3. **Integration test `process_paper`** — use an in-memory SQLite DB + a fixture PDF;
   verify `PaperText` row exists, `job.status == PROCESSED`, `job.stage == PARSE`.
4. **Retry/idempotency test** — run `process_paper` twice on the same job; assert the
   second call returns early (`PROCESSED` guard) without duplicate `PaperText` rows.

---

## 9. Non-Goals (flagged per Architecture Spec)

The following are **out of scope** for KAN-14 and must not be implemented here:

| Feature | Story |
|---------|-------|
| Token-level chunking into `PaperChunk` rows | KAN-15 |
| LLM summarization into `PaperSummary` | KAN-16 |
| Embedding generation | Post-MVP |
| OCR for scanned PDFs | Post-MVP |

These are listed as Non-goals in the Architecture Spec and must not be added to this implementation.

---

## 10. Architect Sign-off

> Reviewed against `docs/design/STEEL_THREAD_DESIGN.md` and `docs/KAN-4-architecture-specification.md`.
> Implementation fits within the FastAPI → Huey → SQLite steel thread.
> No deviations from the approved tech stack.
> PyMuPDF (fitz) confirmed as the extraction library from KAN-37 spike.

**✅ LGTM — @Architect — 2026-03-07**
