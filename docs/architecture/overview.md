# Architecture Overview (Current State)

This document is the concrete, current-state companion to
`docs/volumes/02-architecture.md`'s narrative explanation. Where the volume
explains *why* the architecture is shaped this way, this document is a
factual inventory of what currently exists — keep it updated as the system
changes, or it becomes actively misleading.

## Current Components

| Layer | Component | Status |
|---|---|---|
| Governance | `CLAUDE.md`, `docs/standards/`, `docs/decisions/` | Established |
| Knowledge | `docs/volumes/`, `docs/architecture/`, `docs/references/`, `memory/` | Established |
| Workflow | `prompts/`, `docs/playbooks/`, `templates/` | Established (content minimal, extension points documented) |
| Execution | `src/`, `tests/`, `scripts/`, `.github/workflows/` | Scaffolded; no application code yet |

## What Does Not Exist Yet

- No application language, framework, or runtime has been chosen.
- No database, external service integration, or deployed environment
  exists.
- No CI/CD pipeline is implemented (extension points are documented in
  `docs/volumes/05-workflow-engine.md` and `.github/workflows/README.md`).

## Design Decisions

- This document intentionally says "not yet" rather than omitting sections,
  so a reader can distinguish "not built" from "forgot to document."

## Related Documents

- `docs/volumes/02-architecture.md`
- `docs/volumes/05-workflow-engine.md`
- `memory/technology-choices.md`
