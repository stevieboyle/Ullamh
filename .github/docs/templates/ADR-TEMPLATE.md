# ADR-[STORY-ID]: [Short Decision Title]

**Date:** [DATE]  
**Status:** Proposed | Accepted | Superseded by ADR-[ID]  
**Deciders:** @[agent-or-team]  
**Related story:** [STORY-ID]

---

## Context

Describe the situation that requires a decision. What problem are you solving? What constraints exist? What options were considered?

> Example: "We need a PDF extraction library. The options are PyMuPDF (fitz), pdfplumber, and pypdf. We ran a spike (KAN-37) to evaluate each on our sample corpus."

---

## Decision

State the decision clearly in one or two sentences.

> Example: "We will use PyMuPDF (fitz) as the primary PDF extraction library, using a text-first strategy."

---

## Rationale

Why was this option chosen over the alternatives?

- **[Chosen option]** — key reasons (performance, licence, maintainability, existing use in team, etc.)
- **[Alternative 1]** — why it was rejected
- **[Alternative 2]** — why it was rejected

---

## Consequences

### Positive
- [What good outcomes does this decision produce?]

### Negative / Trade-offs
- [What are the known downsides or limitations?]

### Risks
- [What could go wrong? What would trigger revisiting this decision?]

---

## Revisit Triggers

This decision should be revisited if:
- [Condition 1, e.g. "corpus size exceeds X and performance degrades"]
- [Condition 2]

---

## References

- [Link to spike story, benchmark results, or spec section that informed this decision]
