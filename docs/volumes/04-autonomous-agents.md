# Volume 04 — Autonomous Agents

## Purpose

Define how AI agents (Claude Code sessions, subagents, or other automated
contributors) are expected to operate within AI-EOS: their roles,
responsibilities, boundaries, and coordination model.

## Scope

Covers the conceptual role model for agents working in this repository and
the rules governing autonomy and escalation. Does not cover a specific
runtime/orchestration implementation — no multi-agent runtime is currently
implemented in `src/`; this volume defines the model that any future
implementation must conform to.

## Goals

- Give any agent (human-directed AI session or automated subagent) a clear
  definition of its role, authority, and required checkpoints.
- Prevent uncoordinated or duplicated work when more than one agent
  operates on the repository.
- Define escalation rules so agents ask rather than guess at genuinely
  ambiguous, high-impact, or irreversible decisions.

## Role Model

AI-EOS defines four agent roles. A single session may hold more than one
role depending on the task; these are responsibilities, not job titles.

| Role | Responsibility | Typical output |
|---|---|---|
| **Architect** | Designs structure, evaluates tradeoffs, writes ADRs | ADRs, architecture docs |
| **Implementer** | Writes code/config/docs to a defined spec | Code, tests, documentation |
| **Reviewer** | Checks work against standards and the Review Checklist | Review findings, requested changes |
| **Archivist** | Captures lessons, updates memory, keeps docs in sync | `memory/` updates, doc edits |

Every unit of work should pass through Implementer and Archivist at
minimum; Architect is required for decisions meeting the ADR threshold in
`CLAUDE.md` §14; Reviewer is required before anything is considered Done.

## Design Decisions

- **Roles are responsibilities, not separate systems.** A single Claude
  Code session commonly performs all four roles sequentially within one
  task. This volume does not mandate separate agent processes for each
  role — it mandates that each responsibility is actually discharged.
- **Autonomy has explicit limits.** Per `CLAUDE.md` §10, git pushes require
  explicit approval unless a task has separately authorized autonomous
  commits. This applies equally to any agent, automated or human-directed.
- **Escalate on ambiguity, not on every decision.** An agent should resolve
  what it can reason about from the codebase and `memory/`, and escalate
  (ask the user) only when a decision is genuinely ambiguous, irreversible,
  or outside the scope it was given.
- **No silent scope expansion.** An agent asked to fix a bug does not
  unilaterally refactor unrelated code; if it identifies a worthwhile
  improvement out of scope, it proposes it rather than doing it.

## Best Practices

- State which role you are acting in when the distinction matters (e.g.
  "as Reviewer, this doesn't meet the testing standard because...").
- When multiple agents/sessions might touch the same area concurrently,
  check `memory/roadmap.md` and recent git history before starting, to
  avoid duplicated or conflicting work.
- Archivist responsibilities (updating `memory/`) are not optional cleanup
  — they are part of Definition of Done (`CLAUDE.md` §17).
- When acting as Reviewer, use the checklist in `CLAUDE.md` §12 verbatim
  rather than an ad hoc review.

## Examples

A session is asked to "add rate limiting to the webhook handler":

1. **Architect**: checks `docs/decisions/` for any existing rate-limiting
   decision; if none exists and the approach has real tradeoffs (in-memory
   vs. distributed), writes an ADR.
2. **Implementer**: writes the rate limiter and its tests per
   `docs/standards/testing.md` and `docs/standards/error-handling.md`.
3. **Reviewer**: runs the Review Checklist before declaring the task done.
4. **Archivist**: if a bug was found and fixed along the way, adds a
   `memory/lessons-learned.md` entry; updates `memory/reusable-patterns.md`
   if the rate-limiting approach is likely to be reused.

## Common Mistakes

- Treating "autonomous" as "unsupervised" — autonomy in AI-EOS means
  operating without step-by-step instructions, not without checkpoints or
  approval gates for irreversible actions.
- Skipping the Archivist role because the task "felt small" — small tasks
  are exactly where lessons get lost because no one thinks to record them.
- Multiple agents editing the same document concurrently without checking
  recent history, producing silently overwritten work.
- An agent inventing a new convention instead of checking
  `memory/coding-conventions.md` first.

## Future Improvements

- If a real multi-agent runtime is built under `src/`, extend this volume
  with concrete coordination mechanics (task queues, locking, handoff
  protocols) and link to the implementation.
- Consider a lightweight "session log" convention in `memory/` if
  concurrent-agent conflicts become a recurring problem.

## Related Documents

- `CLAUDE.md` §10, §13, §14 — autonomy limits, planning methodology, decision framework
- `docs/volumes/05-workflow-engine.md` — how agent output flows through the system
- `docs/playbooks/` — step-by-step procedures agents follow for common tasks
