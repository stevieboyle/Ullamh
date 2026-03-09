# Pipeline Parameters Reference
**Version:** 1.0  
**Last updated:** 2026-03-08

This document records all configurable parameters, prompt templates, and threshold values used by the Meitheal processing pipeline. Update this document whenever a parameter is changed.

---

## 1. Stage Overview

The Meitheal processing pipeline progresses through five stages:

```
PENDING → EXTRACTING → SUMMARISING → TAGGING → RESOLVING → COMPLETE
                                                          ↘ FAILED
```

Each stage is implemented as a Huey task in `src/worker/tasks.py` and corresponding service in `src/services/`.

---

## 2. PDF Extraction Parameters

**Service:** `src/services/pdf_extractor.py`  
**Library:** PyMuPDF (`fitz`) — text-first extraction strategy (ADR: KAN-37)

| Parameter | Value | Description |
|---|---|---|
| `MAX_PAGES` | No limit (all pages) | All pages are extracted |
| `EXTRACTION_STRATEGY` | `text-first` | Uses `page.get_text("text")` before any OCR fallback |
| `MIN_TEXT_LENGTH` | 100 characters | Pages with fewer characters are skipped as likely non-text |
| `ENCODING` | UTF-8 | Output encoding for extracted text |

### Notes
- PyMuPDF was selected over `pdfplumber` and `pypdf` based on the KAN-37 spike. See `docs/adr/ADR-KAN-37.md` when created.
- Figures, tables, and mathematical formulae are **not** extracted in MVP (known gap, tracked in tech debt).

---

## 3. Chunking Parameters

**Service:** `src/services/chunker.py`

| Parameter | Value | Description |
|---|---|---|
| `CHUNK_SIZE` | 1000 tokens (approx) | Target size for each text chunk |
| `CHUNK_OVERLAP` | 100 tokens | Overlap between adjacent chunks |
| `SEPARATOR` | `\n\n` | Primary split boundary (paragraph boundary) |

> Note: Chunking is currently used to support the summariser context window. Embedding/RAG chunking is deferred to post-MVP.

---

## 4. Summariser Parameters

**Service:** `src/services/summarizer.py`  
**Activation:** Requires `LLM_API_KEY` to be set. If absent, this stage is skipped and the job proceeds to the next stage with `summary = None`.

### Prompt Template

```
You are a research paper summarizer. Given the following extracted text from a research paper, 
produce a concise, accurate summary of 3-5 sentences that captures:
1. The core research question or objective
2. The key methodology or approach
3. The main findings or contributions

Be precise and use domain-appropriate terminology. Do not include bibliographic information.

Paper text:
{paper_text}
```

### Configuration

| Parameter | Value | Description |
|---|---|---|
| `MAX_INPUT_TOKENS` | 4000 | Text is truncated to this length before sending to LLM |
| `MAX_OUTPUT_TOKENS` | 500 | Maximum summary length |
| `TEMPERATURE` | 0.3 | Low temperature for factual, deterministic output |
| `MODEL` | Provider-dependent (set via `LLM_API_KEY` provider) | LLM model used |

---

## 5. Tagger Parameters

**Service:** `src/services/tagger.py`  
**Activation:** Requires `LLM_API_KEY`. If absent, this stage is skipped and `tags = []`.

### Prompt Template

```
You are a research paper classifier. Given the following extracted text from a research paper,
produce a list of 3-7 concise topic tags.

Rules:
- Tags should be lowercase, hyphen-separated where multi-word (e.g. "machine-learning", "neural-networks")
- Tags should represent the primary research domain and key techniques
- Do not include generic tags like "research", "paper", "study"
- Return tags as a JSON array of strings: ["tag1", "tag2", ...]

Paper text:
{paper_text}
```

### Configuration

| Parameter | Value | Description |
|---|---|---|
| `MAX_INPUT_TOKENS` | 2000 | Text is truncated to this length |
| `MAX_OUTPUT_TOKENS` | 200 | Maximum response length |
| `TEMPERATURE` | 0.1 | Very low temperature for consistent taxonomy |
| `MIN_TAGS` | 1 | Minimum tags required; if LLM returns fewer, stage logs a warning |
| `MAX_TAGS` | 10 | Tags beyond this limit are truncated |

---

## 6. Reference Extractor Parameters

**Service:** `src/services/ref_extractor.py`

| Parameter | Value | Description |
|---|---|---|
| `REFERENCE_SECTION_MARKERS` | `["References", "Bibliography", "Works Cited", "REFERENCES"]` | Section headings used to locate the reference list |
| `MAX_REFERENCES` | 500 | References beyond this limit are discarded with a warning |
| `MIN_REF_LENGTH` | 20 characters | Shorter strings are not treated as references |

---

## 7. Reference Resolver Parameters

**Service:** `src/services/ref_resolver.py`

### Resolution Strategy (Priority Order)

1. **arXiv ID match** — if the extracted reference text contains a recognisable arXiv ID pattern (`\d{4}\.\d{4,5}`)
2. **DOI match** — if the extracted reference text contains a DOI pattern (`10.\d{4,}/\S+`)
3. **Title + Author fuzzy match** — if neither arXiv ID nor DOI is found, use the DOI metadata provider search API with title and first author
4. **Unresolved** — if all strategies fail, the reference is stored with `resolved = False`

### Configuration

| Parameter | Value | Description |
|---|---|---|
| `ARXIV_API_BASE_URL` | `https://export.arxiv.org/api/query` (overridable via env) | arXiv Atom feed endpoint |
| `DOI_RESOLVER_TIMEOUT_SECONDS` | 10 | HTTP timeout for DOI metadata API calls |
| `FUZZY_MATCH_THRESHOLD` | 0.85 | Minimum similarity score (0–1) to accept a title match |
| `MAX_RESOLVER_RETRIES` | 2 | Retries on transient HTTP errors (5xx, timeout) |

---

## 8. Rate Limiter Configuration

**Defined in:** `src/api/upload.py`, `src/api/arxiv.py`  
**Library:** `slowapi` (FastAPI rate limiting)

| Endpoint | Limit | Window | `RateLimiter` variable name |
|---|---|---|---|
| `POST /api/upload` | 10 requests | 60 seconds | `upload_limiter` |
| `POST /api/papers/arxiv` | 10 requests | 60 seconds | `arxiv_limiter` |

> **Test fixture note:** `tests/conftest.py` must reset **both** `upload_limiter` and `arxiv_limiter` between tests. See the `reset_rate_limits` fixture. Failure to reset the arXiv limiter causes intermittent `429` errors in sequential test runs.

---

## 9. Change Log

| Date | Parameter | Old Value | New Value | Reason |
|---|---|---|---|---|
| 2026-03-08 | — | — | — | Initial document created from implementation audit |
