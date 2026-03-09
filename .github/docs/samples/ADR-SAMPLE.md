# ADR-KAN-37: PDF Extraction Library Selection

> **This is a sample document.** It contains project-specific content from the **Meitheal (KAN-4)** project and is provided as a concrete reference showing the level of detail expected in an Architecture Decision Record. Use the template at [`.github/docs/templates/ADR-TEMPLATE.md`](../templates/ADR-TEMPLATE.md) when creating ADRs for a new project.

**Date:** 2026-02-28  
**Status:** Accepted  
**Deciders:** @Architect, @BackendDev, @WorkerDev  
**Related story:** KAN-37 (Technical Decisions Spike)

---

## Context

The Meitheal pipeline needs to extract clean text, title, authors, and abstract from uploaded research-paper PDFs. Papers are often two-column academic layouts with headers, footers, equations, and figures. The extraction library must handle these reliably at the MVP scale (~hundreds of papers).

Three candidate libraries were evaluated during the KAN-37 spike:

| Library | Version | Licence | Notes |
|---|---|---|---|
| PyMuPDF (`fitz`) | 1.24.x | AGPL-3.0 (+ commercial) | C-based, fast, good two-column support |
| pdfplumber | 0.11.x | MIT | Python-based, tabular-layout focus |
| pypdf | 4.x | BSD | Pure Python, basic text extraction |

Each was tested against a 20-paper sample corpus covering single-column, two-column, and figure-heavy layouts.

---

## Decision

We will use **PyMuPDF (fitz)** as the primary PDF extraction library, using a text-first strategy (`page.get_text("text")`).

---

## Rationale

- **PyMuPDF** — Best extraction quality across all layout types in the sample corpus. Handled two-column academic papers correctly in 18/20 test cases. Fast (C bindings). Active maintenance. The AGPL licence is acceptable for a self-hosted, single-user MVP; commercial licence available if needed post-MVP.
- **pdfplumber** — Good for tabular data but produced garbled output on ~40% of two-column papers. Better suited for invoice/form extraction than academic papers.
- **pypdf** — Simplest API and permissive licence, but extraction quality was significantly worse: missed text in 7/20 papers and produced duplicated lines in two-column layouts.

---

## Consequences

### Positive
- High-quality text extraction for academic papers without OCR dependency
- Fast processing (~0.5s per 20-page paper vs ~2s for pdfplumber)
- Well-documented API with active community

### Negative / Trade-offs
- AGPL licence requires careful handling if the project moves to SaaS distribution
- Figures, tables, and mathematical formulae are not extracted as structured data (known MVP gap)
- C extension dependency means the Docker image needs system-level build tools

### Risks
- If the corpus shifts to primarily scanned-image PDFs, PyMuPDF text-first strategy will fail — OCR integration (e.g. Tesseract) would be needed
- AGPL may become a blocker if the project is commercialised without purchasing a commercial licence

---

## Revisit Triggers

This decision should be revisited if:
- The corpus contains >20% scanned-image PDFs requiring OCR
- The project requires a permissive (non-copyleft) licence for distribution
- Extraction quality degrades on a new category of input documents

---

## References

- KAN-37 spike story — Technical Decisions
- Architecture Spec §15 — Technical Decisions (Closed)
- Pipeline Parameters §2 — PDF Extraction Parameters
