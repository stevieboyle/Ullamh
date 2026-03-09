# Pipeline Parameters Reference — [PROJECT_NAME]
**Version:** 1.0  
**Last updated:** [DATE]

> **Usage:** This document records all configurable parameters, prompt templates, and threshold values used by the [PROJECT_NAME] processing pipeline. Intended for AI/ML pipeline projects where prompt text, model settings, and threshold values are first-class configuration. Update this document whenever a parameter is changed — it is the authoritative record of what the pipeline is doing and why.

---

## 1. Stage Overview

Describe the pipeline stages in order.

```
[STAGE_1] → [STAGE_2] → [STAGE_3] → ... → COMPLETE
                                          ↘ FAILED
```

Each stage is implemented as a [task queue] task in `[task-file-path]` and corresponding service in `[services-directory]/`.

---

## 2. [Stage 1] Parameters

**Service:** `[service-file-path]`  
**Library / Tool:** [Library name] — [brief description of approach]

| Parameter | Value | Description |
|---|---|---|
| `[PARAM_1]` | [value] | [What it controls] |
| `[PARAM_2]` | [value] | [What it controls] |
| `[PARAM_3]` | [value] | [What it controls] |

### Notes
- [Any important notes about this stage's implementation or known edge cases]
- [Deferred capabilities or known gaps]

---

## 3. [Stage 2 — e.g. Chunking] Parameters

**Service:** `[service-file-path]`

| Parameter | Value | Description |
|---|---|---|
| `[CHUNK_SIZE]` | [value] | Target size for each chunk |
| `[CHUNK_OVERLAP]` | [value] | Overlap between adjacent chunks |
| `[SEPARATOR]` | `[value]` | Primary split boundary |

---

## 4. [LLM Stage 1 — e.g. Summariser] Parameters

**Service:** `[service-file-path]`  
**Activation:** Requires `[API_KEY_VAR]` to be set. If absent, this stage is skipped and `[field] = None`.

### Prompt Template

```
[Paste the exact prompt template here. Use {variable_name} for interpolated values.]

Input: {[input_variable]}
```

### Configuration

| Parameter | Value | Description |
|---|---|---|
| `MAX_INPUT_TOKENS` | [value] | Input is truncated to this length before sending to LLM |
| `MAX_OUTPUT_TOKENS` | [value] | Maximum output length |
| `TEMPERATURE` | [value] | 0 = deterministic, 1 = creative |
| `MODEL` | [value or "provider-configured"] | LLM model identifier |

---

## 5. [LLM Stage 2 — e.g. Tagger / Classifier] Parameters

**Service:** `[service-file-path]`  
**Activation:** Requires `[API_KEY_VAR]`. If absent, this stage is skipped and `[field] = []`.

### Prompt Template

```
[Paste the exact prompt template here.]

Rules:
- [Rule 1]
- [Rule 2]
- Return output as: [format, e.g. JSON array]

Input: {[input_variable]}
```

### Configuration

| Parameter | Value | Description |
|---|---|---|
| `MAX_INPUT_TOKENS` | [value] | Input truncation length |
| `MAX_OUTPUT_TOKENS` | [value] | Maximum output length |
| `TEMPERATURE` | [value] | Recommended low for consistent classification |
| `MIN_[OUTPUT_ITEMS]` | [value] | Minimum expected outputs |
| `MAX_[OUTPUT_ITEMS]` | [value] | Cap on outputs |

---

## 6. [External Integration] Parameters

**Service:** `[service-file-path]`

### Resolution / Matching Strategy (Priority Order)

1. **[Strategy 1]** — [When this is tried first and what it does]
2. **[Strategy 2]** — [Fallback approach]
3. **[Strategy 3]** — [Second fallback]
4. **Unresolved** — [What happens when all strategies fail]

### Configuration

| Parameter | Value | Description |
|---|---|---|
| `[API_BASE_URL]` | [value] (overridable via env) | External API endpoint |
| `[TIMEOUT_SECONDS]` | [value] | HTTP timeout for external calls |
| `[MATCH_THRESHOLD]` | [value] | Minimum confidence score to accept a match |
| `MAX_RETRIES` | [value] | Retries on transient HTTP errors |

---

## 7. Rate Limiter Configuration

| Endpoint | Limit | Window | Variable Name |
|---|---|---|---|
| `[ENDPOINT_1]` | [N] requests | [X] seconds | `[limiter_var_name]` |
| `[ENDPOINT_2]` | [N] requests | [X] seconds | `[limiter_var_name]` |

> **Test fixture note:** [`tests/conftest.py`](../../tests/conftest.py) must reset **all** rate limiter instances between tests. Missing a reset causes intermittent `429` errors in sequential runs.

---

## 8. Change Log

| Date | Parameter | Old Value | New Value | Reason |
|---|---|---|---|---|
| [DATE] | — | — | — | Initial document created |
