# MVP Functionality Description — [PROJECT_NAME]
**Epic:** [EPIC_ID] — [Epic title]  
**Author:** @Analyst  
**Date:** [DATE]  
**Status:** [In Progress | Complete]

> **Usage:** Written after an MVP or major release is delivered. Describes every shipped feature with enough detail that a new team member, stakeholder, or future developer can understand what was built, how it works, and which Jira tickets delivered it. Update each section as stories are completed. Link every feature to its Jira story.

---

## Table of Contents

- [1. Overview](#1-overview)
- [2. [Feature Area 1]](#2-feature-area-1)
- [3. [Feature Area 2]](#3-feature-area-2)
- [4. [Feature Area 3]](#4-feature-area-3)
- *(add more sections as needed)*
- [N. Infrastructure & DevOps](#n-infrastructure--devops)
- [N+1. Test Coverage](#n1-test-coverage)

---

## 1. Overview

Write a 2–4 sentence summary of the system and what the MVP delivered. State the total number of stories and the current test count.

> **Epic:** [EPIC_ID]  
> **Stories delivered:** [N]  
> **Tests:** [N backend] + [N frontend] + [N E2E] = [total]  
> **Status:** All stories Done / [N] remaining

---

## 2. [Feature Area 1]
*e.g. Paper Ingestion*

### 2.1 [Sub-feature 1] ([STORY-ID])

Describe what this sub-feature does from a user and technical perspective. Include:
- What the user sees/experiences
- How it works technically (key files, endpoint, service)
- Any important constraints (rate limits, size limits, accepted types)

```
Key files:
  src/[layer]/[file].py     — [what it does]
  src/[layer]/[file].py     — [what it does]

Endpoint:
  [VERB] /api/[path]        → [response code]
```

### 2.2 [Sub-feature 2] ([STORY-ID])

[Description as above]

---

## 3. [Feature Area 2]
*e.g. Async Processing Pipeline*

Describe the pipeline as a whole, then document each stage.

```
Status flow: QUEUED → [STAGE_1] → [STAGE_2] → ... → COMPLETE
                                                    ↘ FAILED
```

### 3.1 [Stage 1] ([STORY-ID])
[Technical description]

### 3.2 [Stage 2] ([STORY-ID])
[Technical description]

---

## 4. [Feature Area 3]
*e.g. Job Status Tracking*

[Describe the feature]

---

## [N]. Infrastructure & DevOps
*(covers all enabling/infrastructure stories)*

### [N].1 [Infrastructure Story] ([STORY-ID])
[Description: scaffolding, storage layer, database setup, etc.]

### [N].2 API Configuration & Middleware ([STORY-ID])

| Setting | Value | Notes |
|---|---|---|
| [Config item] | [value] | [notes] |
| CORS origins | [values] | [notes] |
| Rate limits | [values] | [per endpoint] |

---

## [N+1]. Test Coverage

| Layer | Tool | Count | Location |
|---|---|---|---|
| Backend | pytest | [N] | `tests/` |
| Frontend | [e.g. Vitest] | [N] | `web/src/` |
| E2E | Playwright | [N] | `web/tests/` |
| **Total** | | **[N]** | |

### Coverage Highlights
- List the most important test scenarios covered
- Note any known gaps (with links to tech-debt Jira tickets)
