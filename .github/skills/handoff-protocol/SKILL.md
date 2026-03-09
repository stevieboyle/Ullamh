---
name: handoff-protocol
description: Standardized protocol for transferring tasks between specialized agents.
---
# Handoff Protocol
When finishing a task or delegating to another agent, you MUST provide a "Handoff Package" containing:

1. **Status:** Summarize what was completed (e.g., "Database schema for STORY-101 is live").
2. **Artifacts:** List all files created or modified during your turn.
3. **Context for Next:** Explicitly state the next requirement (e.g., "@WorkerDev, the `item_id` is now available in the `items` table for processing").
4. **Blockers:** List any unresolved decisions or technical debt incurred. If none, state "No blockers."
5. **Test Layer Ownership:** State which test layer covers this work and who is responsible:
   - Unit/integration tests: file path(s) in `tests/`
   - Live-HTTP/stack tests: confirm whether a live-stack test was run and by whom
   - E2E Playwright tests: file path(s) in `web/tests/`
   - If a test layer is missing or deferred, explicitly flag it here so @PM can create a follow-on ticket.

# Jira Comment Requirement
For **every handoff**, post a structured comment to the Jira ticket using this template:

```
🔄 Handoff Package — [Agent Name] → [Next Agent]

Status: [what was completed]
Artifacts: [files created/modified]
Context for Next: [explicit next steps for receiving agent]
Blockers: [list or "None"]
Test Layer: [which tests cover this work; flag any gaps]
```

# Role-Specific Triggers
- **@BackendDev → @QAEngineer:** "API endpoints are ready for integration tests in `/tests/api`."
- **@WorkerDev → @UIDev:** "Job status enums are finalized; you can now map the frontend stepper."
- **@Architect → @Any:** "The Architecture Spec has been updated; please pull latest before coding."