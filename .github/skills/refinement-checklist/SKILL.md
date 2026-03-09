---
name: refinement-checklist
description: Use this skill to validate if a Jira story is "Ready for Dev" based on the Architecture Spec.
---
# Refinement Protocol
When a user or another agent asks to "Refine" a story, follow these steps:

## Step 0 — Load the Reference Template
Before refining any story, read the appropriate template from `.github/skills/refinement-checklist/assets/`:

- **Backend / worker stories** → `assets/reference-story.md`
- **Frontend / UI stories** → `assets/reference-ui-story.md`

Select the template based on the story's target layer. Every output story must match the chosen template's structure and level of detail exactly.

## Step 1 — Architecture Check
Cross-reference the story with the project's Architecture Specification (see `.github/TECH_STACK.md` for the approved stack).
- Does it use allowed technologies (see Architecture Spec §2 — Tech Stack)?
- Is it within the "MVP Purpose" (Section 1)?
- Does it align with the processing pipeline stages (Section 5)?

## Step 2 — Definition of Ready (DoR) Validation
Validate all seven template sections are present and complete:

| Section | Minimum Bar |
|---|---|
| **1. User Narrative** | As a / I want / so that |
| **2. Technical Context** | Target layer named; ≥1 concrete file path listed (create or modify) |
| **3. Acceptance Criteria** | ≥3 testable checkboxes with **specific model/route/enum/status-code references**; at least one AC must cover a boundary condition (e.g. rate limit hit, duplicate detected, empty result, malformed input) |
| **4. Definition of Done** | Type safety + test file path + performance target + docs update |
| **5. Edge Cases & Constraints** | ≥2 named failure/constraint scenarios with **explicit HTTP status codes or enum flag values** (e.g. `422`, `429`, `status=FAILED`) |
| **6. Alignment with Architecture Spec** | ≥1 specific section citation (e.g. §5, §11) |
| **7. Cross-Cutting Concerns** | Explicit statement of how the story handles: (a) CORS/auth impact, (b) rate-limiting impact, (c) logging/observability, (d) error propagation to the job status model. If a concern is N/A, state it explicitly. |

Additional checks:
- **Async Flow:** Any story touching async processing must reference the worker/task-queue stages (Section 5).
- **Security:** Any ingestion or storage story must reference the security controls defined in the Architecture Spec (e.g. rate-limiting, access controls).
- **Cross-Cutting Gate:** If the Cross-Cutting Concerns section (§7) is missing or says only "N/A" without justification, the story is ❌ Blocked — cross-cutting concerns must be explicitly reasoned through, not silently skipped.
- **Blocked stories:** If a story depends on an open decision from the Architecture Spec's Open Questions section, it must be flagged ⚠️ BLOCKED and linked to the relevant decision ticket.
- **New Epic Gate:** If this is the first story in a new epic, verify that @Architect has created the Architecture Specification from `.github/docs/templates/ARCHITECTURE_SPECIFICATION-TEMPLATE.md`. A story cannot satisfy §6 (Alignment with Spec) if the spec document does not yet exist — flag to @PM and pause refinement until the spec is in place.

## Step 3 — Output Format
- Produce the full refined story using the reference-story.md template structure.
- Conclude with **Refinement Status**: ✅ Ready for Dev or ❌ Blocked (list exact gaps).
- If updating Jira, write the story description using the template sections as the body.