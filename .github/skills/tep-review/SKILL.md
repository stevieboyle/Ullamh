---
name: tep-review
description: Use this skill when @Architect is asked to review a Technical Execution Plan (TEP). Evaluates completeness, spec alignment, and cross-cutting concerns before approving or requesting changes.
---
# TEP Review Protocol

When an agent asks @Architect to review a TEP, follow these steps.

---

## Step 1 — Load the TEP

Read the TEP at `.github/plans/TEP-[STORY-ID].md`. If the file does not exist, reject the review request and ask the submitting agent to create it from `.github/plans/TEP-TEMPLATE.md` first.

---

## Step 2 — Completeness Check

Verify all 11 sections of the canonical TEP template are present and non-empty:

| Section | Required? | Minimum Bar |
|---|---|---|
| 1. Objective | Mandatory | References the relevant Architecture Spec section |
| 2. Library & Dependency Versions | Mandatory | Lists all new/upgraded deps, or states "None" |
| 3. Target Files | Mandatory | ≥1 concrete file path with create/modify action |
| 4. Pydantic Model Schemas | Mandatory | Shows full field set for new models; delta for modified |
| 5. API Signatures | Mandatory if API routes are added/changed | Route, verb, request/response types, error codes |
| 6. Task & Worker Signatures | Mandatory if async tasks are added/changed | Full task decorator signature |
| 7. Cross-Cutting Concerns | **Always mandatory** | See Cross-Cutting Gate below |
| 8. Acceptance Criteria | Mandatory | ≥3 ACs, at least one boundary-condition AC |
| 9. Test Plan | Mandatory | Lists test files and confirms live-stack test decision |
| 10. Rollback Plan | Mandatory | ≥1 concrete rollback step |
| 11. Open Questions / Risks | Mandatory | Can be empty table if none |

---

## Step 3 — Cross-Cutting Gate (blocking)

Section 7 (Cross-Cutting Concerns) must address all five concerns. For each concern, the TEP must either describe the impact/handling OR explicitly state "N/A" with a one-sentence justification.

| Concern | What to check |
|---|---|
| **CORS** | If a new frontend-facing endpoint is added, are CORS origins covered? Is `CORSMiddleware` already configured for the new path? |
| **Authentication** | Is the endpoint protected or intentionally open? For MVP no-auth is acceptable but must be stated. |
| **Rate Limiting** | If the endpoint is rate-limited, is the `RateLimiter` instance named? Is the test fixture reset confirmed? |
| **Logging** | Are log points at request receipt, task queue, task success, and task failure described? |
| **Error Propagation** | Is the failure → `JobStatus` mapping described? What enum value is set on error? |

If any concern is absent or left blank (not even "N/A"), the review is **blocked**. Do not approve until it is filled in.

---

## Step 4 — Spec Alignment Check

Cross-reference the TEP against the project's Architecture Specification (see `.github/TECH_STACK.md` for the approved stack):
- Are all proposed technologies in the approved tech stack (see Architecture Spec §2)?
- Does the scope fit within the MVP purpose (§1)?
- Does the processing pipeline stage flow (§5) match the proposed task chain?
- Does the story avoid non-goals listed in §1 (embeddings, RAG, graph viz, etc.)?

---

## Step 5 — Delta Update Assessment

Determine whether this story's changes require a Delta Update to `docs/design/STEEL_THREAD_DESIGN.md`:
- **Triggers a Delta Update** if: changes affect >1 file **OR** modify any public API surface (routes, schemas, task signatures)
- **Does not trigger a Delta Update** if: single-file internal refactor with no public contract change

If a Delta Update is required, note it in the review response and track it as an open item.

---

## Step 5b — Documentation Artifact Check

For stories that introduce or modify pipeline services, or change user-visible behaviour, verify that the TEP's Target Files section (§3) includes the appropriate documentation update:

| Story touches... | Expected documentation artifact |
|---|---|
| Any `src/services/` file (pipeline stage) | `docs/pipeline-parameters.md` updated — template: `PIPELINE_PARAMETERS-TEMPLATE.md` |
| New or changed API endpoints | `docs/STARTUP_GUIDE.md` env vars table verified up to date |
| Steel Thread deviation | Delta Update to `docs/design/STEEL_THREAD_DESIGN.md` |
| Spike story closing | `docs/adr/ADR-[STORY-ID].md` created — template: `ADR-TEMPLATE.md` |

If a required documentation artifact is absent from the TEP's Target Files, add it to the "Changes Requested" list. Templates are in `.github/docs/templates/`.

---

## Step 6 — Output Format

Produce a review response in the following format:

```
## TEP Review — [STORY-ID]: [Story Title]
**Reviewer:** @Architect  
**Date:** YYYY-MM-DD  
**Decision:** ✅ LGTM | ❌ Changes Requested

### Findings
[List any issues found, section by section. If LGTM, state "No issues found."]

### Cross-Cutting Concerns
[Confirm each of the 5 concerns was addressed, or list which are missing.]

### Delta Update Required?
[Yes / No — reason]

### Next Steps
[If approved: "Coding may begin." If changes requested: list specific items to fix.]
```

Post this review as a comment on the Jira ticket and update the TEP file's "Architect Review" status field.
