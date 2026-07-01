# Architecture Decision Records (ADRs)

This directory records significant, hard-to-reverse architectural
decisions and the reasoning behind them. See `CLAUDE.md` §14 for when an
ADR is required.

## Process

1. Copy `templates/adr-template.md` to `docs/decisions/NNNN-short-title.md`,
   using the next sequential four-digit number (never reused).
2. Fill in Context, Decision, Alternatives Considered, and Consequences.
3. Add a row to `memory/architecture-decisions.md`.
4. If the ADR supersedes a previous one, link both directions and update
   the superseded ADR's status.

## Index

See `memory/architecture-decisions.md` for the fast-lookup index. This
directory holds full reasoning; that file holds one-line summaries.

## Related Documents

- `CLAUDE.md` §14 — Decision-Making Framework
- `templates/adr-template.md`
- `memory/architecture-decisions.md`
