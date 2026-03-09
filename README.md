# Ullamh

A reusable project-management framework for AI-assisted software development teams.

## What's Inside

The `.github/` directory contains a complete, project-agnostic toolkit:

- **Agent definitions** — specialised roles (Analyst, Architect, BackendDev, UIDev, WorkerDev, QAEngineer, ProjectManager) with instructions for AI coding assistants
- **Skills** — reusable workflows for story refinement, TEP review, handoff protocols, and more
- **Document templates** — Architecture Spec, Dependency Plan, Steel Thread Design, Operations Runbook, Retrospective, ADR, and others
- **Populated samples** — real-world examples from a completed MVP project showing what each template looks like when filled in
- **Process documentation** — Operating Procedures, Tech Stack manifest, and Definition of Done

## Getting Started

1. Fork or clone this repo
2. Update `.github/TECH_STACK.md` with your project's technology choices
3. Follow the [Operating Procedures](.github/docs/OPERATING_PROCEDURES.md) to set up your first epic

## Structure

```
.github/
├── agents/          # Agent role definitions
├── docs/
│   ├── samples/     # Populated reference documents
│   └── templates/   # Blank document templates
├── plans/           # TEP template and plans directory
├── skills/          # Reusable workflow skills
├── copilot-instructions.md   # Global standards
├── TECH_STACK.md              # Technology manifest
└── AGENTS.md                  # Agent registry
```

## Licence

See [LICENSE](LICENSE) for details.
