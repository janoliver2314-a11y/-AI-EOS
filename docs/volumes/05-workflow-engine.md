# Volume 05 — Workflow Engine

## Purpose

Describe how work moves through AI-EOS from an idea or request to a
completed, documented, memory-updated change — and where automation attaches
to that flow.

## Scope

Covers the conceptual workflow (not a specific software "workflow engine"
product) and the extension points where CI/CD, MCP servers, and other
automation are designed to attach. No automation is fully implemented yet;
this volume defines clean seams for it, per `CLAUDE.md`'s instruction to
design extension points rather than ship unfinished integrations.

## Goals

- Make the path from "request" to "done" explicit and repeatable.
- Define where automation should attach without requiring it to exist yet.
- Ensure every workflow step has an owner (a role from Volume 04) and a
  durable artifact (a file that outlives the session).

## The Core Workflow

```
 Request/Idea
      │
      ▼
 1. Clarify & Survey  ──── check memory/, docs/decisions/, docs/standards/
      │
      ▼
 2. Plan               ──── for non-trivial work; write down the approach
      │
      ▼
 3. Implement           ──── code/docs/config change, following standards
      │
      ▼
 4. Test                ──── new/changed behavior has a failing-then-passing test
      │
      ▼
 5. Review              ──── CLAUDE.md §12 checklist
      │
      ▼
 6. Record              ──── memory/ updated (lessons, patterns, roadmap, decisions)
      │
      ▼
 7. Approve & Ship      ──── explicit approval before commit/push (CLAUDE.md §10)
```

Steps 1–6 can repeat in small loops for incremental work; step 7 is a gate,
not a formality — it is the point where the change becomes part of the
project's permanent history.

## Automation Extension Points

These are **designed, not implemented**. Each has a defined seam so future
automation can attach without redesigning the workflow:

| Extension point | Attaches at step | Where it lives |
|---|---|---|
| CI (lint/test/build) | 4 (Test), 5 (Review) | `.github/workflows/` |
| MCP servers (external tool access) | 1, 3 (Survey, Implement) | Documented per-server in `docs/references/` when added |
| Documentation generation | 6 (Record) | `scripts/` (e.g. an index generator for `docs/`) |
| Knowledge indexing / retrieval | 1 (Clarify & Survey) | `scripts/`, consuming `memory/` and `docs/` |
| n8n or external workflow automation | 6, 7 (Record, Ship) | Triggered via webhook from `.github/workflows/`; configuration documented, not committed with secrets |
| Deployment automation | after 7 (Ship) | `.github/workflows/`, detailed in `docs/volumes/08-deployment.md` |

## Design Decisions

- **The workflow is sequential in description but iterative in practice.**
  Small tasks compress steps 1–6 into minutes; the shape still applies so
  nothing is silently skipped (especially step 6, Record).
- **Automation is additive, not load-bearing, until proven.** A newly added
  automation extension point starts as opt-in tooling; it does not become a
  hard gate (e.g., a required CI check) until it has demonstrated it
  produces more value than friction.
- **No partial integrations are merged.** An MCP server, CI workflow, or
  script that is half-configured is worse than none — it creates false
  confidence. See `docs/standards/dependency-management.md`.
- **Extension points are documented before they are built**, so the first
  implementation has a spec to satisfy rather than inventing the contract
  ad hoc.

## Best Practices

- When adding automation, first find the row in the Extension Points table
  above (or add one) before writing implementation.
- Keep automation scripts small and single-purpose; compose them rather
  than writing monolithic pipeline scripts.
- Any automation that can push, deploy, or modify external systems must be
  reviewed as security-sensitive per `docs/standards/security.md`.
- Document the trigger, inputs, outputs, and failure mode of every
  automation before merging it — a workflow with an undocumented failure
  mode is a future incident.

## Examples

Adding GitHub Actions CI for linting:

1. Check the table above: CI attaches at steps 4–5.
2. Add `.github/workflows/lint.yml` with a clear, single responsibility.
3. Document what it checks and how to run the same check locally in
   `docs/standards/testing.md` or a dedicated playbook.
4. Start as a non-blocking check; promote to required once it has run
   clean for a reasonable period.

## Common Mistakes

- Building a CI/CD pipeline before there is code for it to build or test —
  automation with nothing to automate is speculative complexity.
- Wiring an MCP server or webhook integration with credentials committed to
  the repository instead of documented as required environment variables.
- Skipping step 6 (Record) because automation "will catch it later" — no
  automation currently populates `memory/`; that step is manual until an
  extension point is actually built for it.
- Treating an extension point placeholder as permission to build the full
  integration immediately — design the seam, implement only what's needed
  now.

## Future Improvements

- Build the first real extension: a documentation index generator in
  `scripts/` that keeps `docs/README.md` and `docs/volumes/10-master-reference.md`
  cross-links accurate.
- Once `src/` has real application code, add a minimal CI workflow (lint +
  test) as the first `.github/workflows/` implementation.

## Related Documents

- `docs/volumes/02-architecture.md` — Workflow/Execution layer boundary
- `docs/volumes/04-autonomous-agents.md` — role ownership of each workflow step
- `docs/volumes/08-deployment.md` — what happens after step 7
- `scripts/README.md`, `.github/workflows/README.md` — concrete extension point status
