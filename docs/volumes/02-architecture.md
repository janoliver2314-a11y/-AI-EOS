# Volume 02 — Architecture

## Purpose

Describe the structural components of AI-EOS, how they relate, and the
principles that govern how the system grows.

## Scope

Covers the repository's structural architecture (documentation system,
memory system, workflow/automation surfaces) and the conventions for how
application code under `src/` should be organized as it is built. Does not
cover specific engineering standards (Volume 06) or the autonomous agent
role model (Volume 04) beyond how they attach to this architecture.

## Goals

- Provide a single diageuatable mental model of how AI-EOS's parts fit
  together.
- Define stable extension points so new capabilities (agents, automation,
  application code) have an obvious place to attach.
- Prevent architectural drift by making the intended shape explicit and
  reviewable.

## System Overview

AI-EOS is composed of four cooperating layers:

```
┌─────────────────────────────────────────────────────────────┐
│  1. Governance Layer                                         │
│     CLAUDE.md, docs/standards/, docs/decisions/ (ADRs)        │
│     — defines the rules everything else operates under        │
├─────────────────────────────────────────────────────────────┤
│  2. Knowledge Layer                                           │
│     docs/volumes/, docs/architecture/, docs/references/,      │
│     memory/ — the durable, accumulated understanding of the   │
│     system and its history                                    │
├─────────────────────────────────────────────────────────────┤
│  3. Workflow Layer                                             │
│     prompts/, docs/playbooks/, templates/ — repeatable         │
│     procedures that turn intent into action consistently       │
├─────────────────────────────────────────────────────────────┤
│  4. Execution Layer                                            │
│     src/, tests/, scripts/, .github/workflows/ — the actual    │
│     application code and automation that does the work         │
└─────────────────────────────────────────────────────────────┘
```

Data/decisions generally flow downward (governance shapes knowledge shapes
workflow shapes execution) and lessons flow upward (execution produces
lessons that update knowledge, which may in turn update governance).

## Design Decisions

- **Layered, not layered-and-locked.** Layers describe intended dependency
  direction, not a hard technical enforcement mechanism — this is a
  documentation and process architecture, not a runtime one.
- **`src/` is intentionally unopinionated about a specific framework at
  this stage.** AI-EOS does not yet mandate a language or framework for
  application code; that decision is deferred to an ADR once a concrete
  application need exists (see `docs/decisions/`). Building the
  documentation and standards system first, and the application second,
  avoids standards that are accidentally coupled to a specific stack.
- **Memory is separate from documentation.** `docs/` describes the system
  as it is intended to work; `memory/` records what has actually happened
  (decisions made, bugs fixed, ideas deferred). Conflating the two makes
  both harder to trust.
- **Automation extension points are designed before they are implemented.**
  See Volume 05 and `docs/standards/dependency-management.md` — this
  avoids half-finished integrations that are worse than no integration.

## Best Practices

- When adding a new top-level capability, first ask which layer it belongs
  to and whether an existing directory already covers it.
- Keep `docs/architecture/` for concrete, current-state descriptions
  (diagrams, component inventories); keep `docs/volumes/` for narrative,
  reasoning-heavy content. If a fact belongs in both, `docs/architecture/`
  is the source of truth and volumes link to it.
- Any change that alters this layered model is architecturally significant
  enough to warrant an ADR.

## Examples

Adding a new automated documentation generator:

- It is automation → belongs conceptually to the Workflow/Execution
  boundary.
- Its trigger and configuration live in `.github/workflows/` or `scripts/`
  (Execution Layer).
- Its purpose and design are documented in `docs/volumes/05-workflow-engine.md`
  and, if it changes how documentation is authored, referenced from
  `docs/standards/documentation.md` (Governance Layer).

## Common Mistakes

- Placing narrative "why" content in `docs/architecture/` where it will be
  mistaken for a current-state fact sheet.
- Building execution-layer automation before the workflow it automates has
  been documented — this produces automation nobody can safely modify.
- Skipping an ADR for a structural change because "it's just a folder
  rename" — structural changes affect every future contributor's mental
  model and are cheap to document, expensive to leave undocumented.

## Future Improvements

- Once `src/` has real application code, add a component diagram under
  `docs/architecture/` reflecting the actual runtime architecture.
- Evaluate whether `docs/diagrams/` should standardize on Mermaid (renders
  natively in GitHub) as the default diagram format — see
  `docs/diagrams/README.md`.

## Related Documents

- `docs/volumes/01-foundation.md` — mission and philosophy this architecture serves
- `docs/volumes/03-memory-system.md` — the Knowledge Layer's memory component
- `docs/volumes/05-workflow-engine.md` — the Workflow/Execution boundary in detail
- `docs/architecture/overview.md` — concrete component inventory
- `docs/decisions/` — architecture decision records
