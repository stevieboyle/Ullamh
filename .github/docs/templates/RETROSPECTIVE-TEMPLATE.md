# [EPIC_ID / PROJECT_NAME] Sprint Retrospective
**Date:** [DATE]  
**Facilitated by:** @ProjectManager  
**Participants:** [List agents/team members]  
**Scope:** [All stories in sprint / epic]

> **Usage:** Fill in during or immediately after sprint/project completion. Collect input from each specialist/team role individually, then synthesise into recommendations. The goal is actionable process improvements, not blame.

---

## Executive Summary

Write 2–4 sentences covering: what was delivered, the test count, and the top 3–6 themes that emerged.

Enumerate the themes:
1. **[Theme 1]** — brief description
2. **[Theme 2]** — brief description
3. **[Theme 3]** — brief description

---

## Retrospective Inputs by Specialist / Role

> For each role below, remove roles that don't apply to your team and add any missing ones.

### @Analyst
- [What went well from a requirements/refinement perspective?]
- [What was missing from acceptance criteria?]
- [Any cross-cutting concerns that were missed in refinement?]
- [Anything that should change in the refinement process?]

### @Architect
- [Was the spec kept up to date? Were Delta Updates applied properly?]
- [Were there cross-cutting concerns (CORS, auth, rate limiting) that fell through the cracks?]
- [Were TEPs reviewed and approved before coding began?]
- [What improvements to the TEP or spec process would help?]

### @BackendDev
- [What implementation patterns worked well?]
- [What caused rework or unexpected friction?]
- [Were the instructions/constraints in the agent file accurate?]
- [Any tooling or environment issues?]

### @WorkerDev
- [What async/pipeline patterns worked well?]
- [Were fixtures and mocks in place before external API coding began?]
- [Any issues with idempotency or error handling?]

### @UIDev
- [What frontend patterns worked well?]
- [Any port, test configuration, or toolchain surprises?]
- [Were component boundaries (Server vs Client) clear?]

### @QAEngineer
- [Were all rate limiter / fixture resets in place?]
- [Was a live-stack test layer used for API stories?]
- [Were there test file naming or test runner conflicts?]
- [What would have caught bugs earlier?]

---

## Recommendations

Group recommendations by category. Assign a priority (HIGH / MEDIUM / LOW) to each.

### Category 1 — [e.g. Process / Instructions]

| ID | Priority | Recommendation | Target File |
|---|---|---|---|
| REC-01 | HIGH | [Specific, actionable recommendation] | [File to change] |
| REC-02 | MEDIUM | [Recommendation] | [File to change] |

### Category 2 — [e.g. Templates & Artifacts]

| ID | Priority | Recommendation | Target File |
|---|---|---|---|
| REC-03 | HIGH | [Recommendation] | [File to change] |

### Category 3 — [e.g. Agent Instructions]

| ID | Priority | Recommendation | Target File |
|---|---|---|---|
| REC-04 | MEDIUM | [Recommendation] | [Agent file] |

### Category 4 — [e.g. Technical Debt]

| ID | Priority | Recommendation | Target File |
|---|---|---|---|
| REC-05 | MEDIUM | Create Jira backlog tickets for [N] tech debt items | Jira |

---

## Priority Matrix

| Priority | Count | Recommendation IDs |
|---|---|---|
| HIGH | [N] | REC-01, REC-... |
| MEDIUM | [N] | REC-02, REC-... |
| LOW | [N] | REC-... |

---

## Tech Debt Inventory

List any deferred items or known technical debt that should become Jira backlog tickets. These should be created by @PM as follow-on stories.

1. **[Debt item 1]** — [description, impact, and suggested priority]
2. **[Debt item 2]** — [description]
3. *(continue as needed)*

---

## Implementation Status

| Recommendation | Status |
|---|---|
| REC-01 through REC-[N] | ☐ Pending / ✅ Implemented |

---

*Retrospective facilitated by @ProjectManager on [DATE]. Inputs collected from [list participants].*
