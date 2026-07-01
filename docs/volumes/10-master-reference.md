# Volume 10 — Master Reference

## Purpose

Serve as the single index and quick-reference point across all of AI-EOS's
documentation, so any reader can find the right document in one hop.

## Scope

This volume indexes and briefly summarizes every other volume and the major
supporting document sets (`docs/standards/`, `memory/`, `docs/decisions/`).
It intentionally contains no new normative content of its own — if a rule
seems to belong here, it belongs in the document this volume links to
instead.

## Goals

- Provide a single entry point for navigating the entire documentation
  system.
- Keep cross-references between volumes accurate as the documentation set
  grows.
- Serve as a lightweight glossary/map for onboarding a new session or
  contributor in one read.

## Volume Index

| # | Volume | One-line summary |
|---|---|---|
| 01 | [Foundation](01-foundation.md) | Mission, philosophy, and the problem AI-EOS solves |
| 02 | [Architecture](02-architecture.md) | The four-layer structural model of the system |
| 03 | [Memory System](03-memory-system.md) | What `memory/` stores and how it's retrieved |
| 04 | [Autonomous Agents](04-autonomous-agents.md) | Agent roles, autonomy limits, escalation rules |
| 05 | [Workflow Engine](05-workflow-engine.md) | The request-to-done workflow and automation seams |
| 06 | [Engineering Standards](06-engineering-standards.md) | Index and rationale for `docs/standards/` |
| 07 | [Security](07-security.md) | Threat model, security principles, disclosure process |
| 08 | [Deployment](08-deployment.md) | Environment strategy and release principles |
| 09 | [Business Strategy](09-business-strategy.md) | Vision, audience, and strategic direction |
| 10 | Master Reference | This document |

## Quick Reference by Task

| I want to... | Go to |
|---|---|
| Understand the project's rules before doing anything | `CLAUDE.md` |
| Understand *why* AI-EOS exists | Volume 01 |
| Understand how the pieces fit together | Volume 02 |
| Find a prior decision or lesson | `memory/`, Volume 03 |
| Know how an AI session should behave | Volume 04 |
| Know the step-by-step process for a change | Volume 05, `docs/playbooks/` |
| Find a specific coding rule | `docs/standards/`, Volume 06 |
| Handle a security-sensitive change | Volume 07, `docs/standards/security.md` |
| Plan a release | Volume 08 |
| Understand project direction and priorities | Volume 09, `memory/roadmap.md` |
| Record an architectural decision | `docs/decisions/`, `templates/adr-template.md` |
| Use a reusable prompt for AI-assisted work | `prompts/` |

## Design Decisions

- **This volume has no normative content of its own** — deliberately, to
  avoid becoming a second, competing source of truth. It only summarizes
  and links.
- **The table-based format is intentional** — a master reference should be
  scannable in seconds, not read start to finish.

## Best Practices

- When adding a new volume, standard, or major document, add a row here in
  the same commit.
- If this document and the document it links to disagree, the linked
  document wins — update this index to match, immediately.

## Examples

A new Claude Code session with no prior context should be able to: read
`CLAUDE.md`, then this volume, then follow links to whatever specific
document its task requires — without needing to guess at the repository's
structure.

## Common Mistakes

- Letting this index go stale after adding a new document elsewhere —
  treat an out-of-date master reference as a broken link, not a minor
  omission.
- Adding substantive content here instead of in the relevant volume,
  defeating the purpose of having a single source of truth per topic.

## Future Improvements

- Once `scripts/` automation exists (Volume 05), consider generating this
  table automatically from document front matter to guarantee it never
  drifts.

## Related Documents

- All documents in `docs/volumes/`, `docs/standards/`, and `memory/` are, by
  definition, related to this index.
