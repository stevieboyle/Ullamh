# Document Samples

This folder contains real-world examples of each document type taken from the **Meitheal / KAN-4** project. They are provided as concrete references to help you understand what each template looks like when fully populated.

These are **not templates** — they contain project-specific content and should not be used as starting points. Use the files in [`../templates/`](../templates/) when creating documents for a new project.

---

## Samples Index

| Sample File | Source Document | Template |
|---|---|---|
| [ADR-SAMPLE.md](ADR-SAMPLE.md) | Created from KAN-37 spike decisions | [ADR-TEMPLATE.md](../templates/ADR-TEMPLATE.md) |
| [ARCHITECTURE_GUIDE-SAMPLE.md](ARCHITECTURE_GUIDE-SAMPLE.md) | `docs/ARCHITECTURE_GUIDE.md` | [ARCHITECTURE_GUIDE-TEMPLATE.md](../templates/ARCHITECTURE_GUIDE-TEMPLATE.md) |
| [ARCHITECTURE_SPECIFICATION-SAMPLE.md](ARCHITECTURE_SPECIFICATION-SAMPLE.md) | `docs/KAN-4-architecture-specification.md` | [ARCHITECTURE_SPECIFICATION-TEMPLATE.md](../templates/ARCHITECTURE_SPECIFICATION-TEMPLATE.md) |
| [DEPENDENCY_PLAN-SAMPLE.md](DEPENDENCY_PLAN-SAMPLE.md) | `docs/KAN-4-dependency-plan.md` | [DEPENDENCY_PLAN-TEMPLATE.md](../templates/DEPENDENCY_PLAN-TEMPLATE.md) |
| [MVP_FUNCTIONALITY-SAMPLE.md](MVP_FUNCTIONALITY-SAMPLE.md) | `docs/MVP-FUNCTIONALITY.md` | [MVP_FUNCTIONALITY-TEMPLATE.md](../templates/MVP_FUNCTIONALITY-TEMPLATE.md) |
| [OPERATIONS_RUNBOOK-SAMPLE.md](OPERATIONS_RUNBOOK-SAMPLE.md) | `docs/OPERATIONS_RUNBOOK.md` | [OPERATIONS_RUNBOOK-TEMPLATE.md](../templates/OPERATIONS_RUNBOOK-TEMPLATE.md) |
| [PIPELINE_PARAMETERS-SAMPLE.md](PIPELINE_PARAMETERS-SAMPLE.md) | `docs/pipeline-parameters.md` | [PIPELINE_PARAMETERS-TEMPLATE.md](../templates/PIPELINE_PARAMETERS-TEMPLATE.md) |
| [REFERENCE_STORY-SAMPLE.md](REFERENCE_STORY-SAMPLE.md) | `.github/skills/refinement-checklist/assets/reference-story.md` (pre-generalisation) | [reference-story.md](../../../.github/skills/refinement-checklist/assets/reference-story.md) |
| [REFERENCE_UI_STORY-SAMPLE.md](REFERENCE_UI_STORY-SAMPLE.md) | `.github/skills/refinement-checklist/assets/reference-ui-story.md` (pre-generalisation) | [reference-ui-story.md](../../../.github/skills/refinement-checklist/assets/reference-ui-story.md) |
| [RETROSPECTIVE-SAMPLE.md](RETROSPECTIVE-SAMPLE.md) | `docs/RETROSPECTIVE-2026-03-08.md` | [RETROSPECTIVE-TEMPLATE.md](../templates/RETROSPECTIVE-TEMPLATE.md) |
| [STARTUP_GUIDE-SAMPLE.md](STARTUP_GUIDE-SAMPLE.md) | `docs/STARTUP_GUIDE.md` | [STARTUP_GUIDE-TEMPLATE.md](../templates/STARTUP_GUIDE-TEMPLATE.md) |
| [STEEL_THREAD_DESIGN-SAMPLE.md](STEEL_THREAD_DESIGN-SAMPLE.md) | `docs/design/STEEL_THREAD_DESIGN.md` | [STEEL_THREAD_DESIGN-TEMPLATE.md](../templates/STEEL_THREAD_DESIGN-TEMPLATE.md) |
| [TEP-SAMPLE.md](TEP-SAMPLE.md) | `.github/plans/TEP-KAN-14.md` | [TEP-TEMPLATE.md](../../plans/TEP-TEMPLATE.md) |
| [USER_GUIDE-SAMPLE.md](USER_GUIDE-SAMPLE.md) | `docs/USER_GUIDE.md` | [USER_GUIDE-TEMPLATE.md](../templates/USER_GUIDE-TEMPLATE.md) |

---

## Notes

- All samples are from **Sprint 1–3** of the KAN-4 epic (paper ingestion pipeline, FastAPI + Huey + SQLite + Next.js stack).
- The **architecture guide** sample (~700 lines) is the richest document — 13 sections with Mermaid diagrams covering system overview, data flows, worker pipeline, ER diagram, and inter-component data contracts.
- The **architecture specification** sample is the foundational spec document with 15 sections covering purpose, tech stack, data model, API surface, and technical decisions.
- The **dependency plan** sample shows a full story-level dependency DAG with Mermaid diagram, steel thread identification, and layer-by-layer breakdown.
- The **MVP functionality** sample documents every delivered feature across 10 sections including test coverage (118 tests).
- The **user guide** sample shows end-user documentation with step-by-step guides for every feature.
- The **retrospective** sample covers a full post-MVP retro with 25 prioritised recommendations — this retrospective drove creation of many process improvements now in the `.github/` framework.
- The **steel thread design** sample shows how a Mermaid sequence diagram evolves through Delta Updates as implementation deviates from initial design.
- The **operations runbook** sample shows the full runbook format with docker-compose deployment, backup procedures, incident response, and capacity planning.
- The **ADR** sample documents the KAN-37 spike decision on PDF extraction library selection — shows the expected depth of context, rationale, and revisit triggers.
- The **TEP** sample (KAN-14: PDF text extraction) shows a fully approved plan with scope, library versions, file paths, class APIs, DB write summary, edge cases, testing notes, and architect sign-off.
- The **pipeline parameters** sample records all configurable values (chunking sizes, LLM prompt templates, resolver thresholds, rate limits) for the 5-stage processing pipeline.
