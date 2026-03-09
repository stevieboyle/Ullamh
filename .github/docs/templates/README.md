# Document Templates

This folder contains generic templates for every document type used across projects in this repository. When starting a new project or epic, copy the relevant templates into your project's `docs/` directory and fill in the placeholders.

## Placeholder Conventions

| Placeholder | Replace with |
|---|---|
| `[PROJECT_NAME]` | The name of your application (e.g. "Meitheal") |
| `[EPIC_ID]` | The primary Jira epic ID (e.g. "KAN-4") |
| `[AUTHOR]` | The agent or team member who owns this document |
| `[DATE]` | Document creation or last-updated date (YYYY-MM-DD) |
| `[TECH_STACK]` | Your chosen technologies |
| `[STORY_ID]` | A specific Jira story/ticket ID |

## Available Templates

| Template File | Use When | Output Location |
|---|---|---|
| `ADR-TEMPLATE.md` | Recording an architecture decision | `docs/adr/ADR-[STORY-ID].md` |
| `ARCHITECTURE_GUIDE-TEMPLATE.md` | Documenting the as-built architecture | `docs/ARCHITECTURE_GUIDE.md` |
| `ARCHITECTURE_SPECIFICATION-TEMPLATE.md` | Defining the target architecture before build | `docs/[EPIC-ID]-architecture-specification.md` |
| `DEPENDENCY_PLAN-TEMPLATE.md` | Ordering stories and mapping build dependencies | `docs/[EPIC-ID]-dependency-plan.md` |
| `MVP_FUNCTIONALITY-TEMPLATE.md` | Describing what was built in an MVP/release | `docs/MVP-FUNCTIONALITY.md` |
| `OPERATIONS_RUNBOOK-TEMPLATE.md` | Operating and maintaining a deployed system | `docs/OPERATIONS_RUNBOOK.md` |
| `PIPELINE_PARAMETERS-TEMPLATE.md` | Documenting AI/ML pipeline configuration | `docs/pipeline-parameters.md` |
| `RETROSPECTIVE-TEMPLATE.md` | Running a sprint or project retrospective | `docs/RETROSPECTIVE-[DATE].md` |
| `STARTUP_GUIDE-TEMPLATE.md` | Developer and operator on-boarding | `docs/STARTUP_GUIDE.md` |
| `STEEL_THREAD_DESIGN-TEMPLATE.md` | Defining the minimum end-to-end flow | `docs/design/STEEL_THREAD_DESIGN.md` |
| `USER_GUIDE-TEMPLATE.md` | End-user documentation | `docs/USER_GUIDE.md` |

## Sample Documents

See [`../samples/`](../samples/) for completed examples from the Meitheal (KAN-4) project. These show the level of detail expected in each document type.
