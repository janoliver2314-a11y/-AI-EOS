# ADR 0001: Record Architecture Decisions

- **Status**: Accepted
- **Date**: 2026-07-01

## Context

AI-EOS is designed around the principle that context loss between sessions
(human or AI) is the primary long-term risk to engineering quality
(`docs/volumes/01-foundation.md`). Architectural decisions — especially
ones that are expensive to reverse or that establish a convention future
work must follow — need a durable record of not just *what* was decided
but *why*, including what alternatives were rejected and why.

Without such a record, decisions get silently re-litigated, contradicted,
or forgotten, and future contributors (or AI sessions) waste effort
rediscovering reasoning that already existed.

## Decision

Adopt Architecture Decision Records (ADRs) as the mechanism for recording
significant decisions:

- ADRs live in `docs/decisions/`, one file per decision, numbered
  sequentially (`NNNN-short-title.md`), numbers never reused.
- An ADR is required whenever a decision (per `CLAUDE.md` §14): is
  expensive or awkward to reverse; affects more than one module/domain;
  chooses between two or more viable technical approaches; or establishes
  a convention future work is expected to follow.
- Each ADR records Context, Decision, Alternatives Considered, and
  Consequences.
- ADRs are never deleted. A changed decision produces a new ADR that
  supersedes the old one, with links in both directions.
- `memory/architecture-decisions.md` maintains a fast-lookup index of all
  ADRs with one-line summaries.

## Alternatives Considered

- **No formal decision record; rely on commit messages and PR
  descriptions.** Rejected: commit/PR history is not organized for
  decision lookup, gets buried among unrelated changes, and doesn't
  clearly capture rejected alternatives.
- **A single running "decisions log" document.** Rejected: a single
  growing file becomes hard to link to precisely and hard to review
  incrementally, the same problem the ten-volume documentation split
  (Volume 06) avoids for narrative docs.
- **Decisions recorded only in `memory/`.** Rejected: `memory/` is designed
  as a fast-scan index (Volume 03), not a place for full reasoning —
  conflating the two would make `memory/` slow to scan again.

## Consequences

- Every significant decision has a discoverable, permanent record,
  improving onboarding speed and reducing re-litigation.
- Adds a small amount of process overhead for genuinely significant
  decisions — mitigated by scoping ADRs to decisions that meet the
  threshold in `CLAUDE.md` §14, not every code change.
- Requires discipline to keep `memory/architecture-decisions.md` in sync
  with `docs/decisions/`; treated as part of the same change, not a
  follow-up task.

## Related Documents

- `CLAUDE.md` §14
- `docs/decisions/README.md`
- `templates/adr-template.md`
- `memory/architecture-decisions.md`
