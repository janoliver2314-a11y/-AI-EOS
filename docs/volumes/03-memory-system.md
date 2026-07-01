# Volume 03 — Memory System

## Purpose

Define what "project memory" means in AI-EOS, what it stores, and how it is
retrieved and kept trustworthy over time.

## Scope

Covers the `memory/` directory: its files, their purpose, update
responsibilities, and the retrieval discipline expected before new work
begins. Does not cover long-form conceptual documentation (`docs/volumes/`)
or formal architecture decisions (`docs/decisions/`), though memory
summarizes and links to both.

## Goals

- Make prior decisions, conventions, and lessons cheap to find before new
  work starts, so mistakes are not repeated and conventions are not
  reinvented inconsistently.
- Keep memory lightweight enough that it is actually maintained, rather
  than becoming another form of documentation debt.
- Provide a clear contract for what belongs in memory versus in `docs/`.

## What Memory Stores

| File | Contents |
|---|---|
| `memory/architecture-decisions.md` | Running index of ADRs with one-line summaries and links |
| `memory/coding-conventions.md` | Concrete, current conventions in force (language/tool specific) |
| `memory/technology-choices.md` | What technologies were chosen and why, at a glance |
| `memory/lessons-learned.md` | Root-cause entries for bugs and mistakes, per `CLAUDE.md` §15 |
| `memory/roadmap.md` | Near/mid-term plan, in priority order |
| `memory/future-ideas.md` | Deferred ideas not yet prioritized — a parking lot, not a commitment |
| `memory/reusable-patterns.md` | Patterns proven useful more than once, with a canonical example |

## Design Decisions

- **Memory is a working document set, not an archive.** Entries are kept
  current; superseded entries are marked as such rather than deleted, so
  history remains visible.
- **Memory is index-first.** Each file is organized for fast scanning (most
  recent or most relevant first, short entries, consistent structure) —
  not for narrative reading.
- **Lessons are structured, not freeform.** Every `lessons-learned.md`
  entry follows the fixed shape in `CLAUDE.md` §15 (Root Cause, Why It
  Happened, Solution, Preventive Rule, Similar Situations) so entries stay
  scannable and comparable.
- **Memory does not duplicate ADRs; it indexes them.** The full reasoning
  for a decision lives in `docs/decisions/`; `memory/architecture-decisions.md`
  is a fast-lookup summary.

## Best Practices

- Before writing new code in an unfamiliar area, search `memory/lessons-learned.md`
  and `memory/reusable-patterns.md` first — this is a required step in
  `CLAUDE.md` §15, not an optional nicety.
- Update `memory/roadmap.md` and `memory/future-ideas.md` whenever priorities
  shift, not just when starting a new project phase.
- Keep entries short. A lesson entry that takes ten minutes to read will not
  be read. Aim for a paragraph per section, not a page.
- When a convention in `memory/coding-conventions.md` is contradicted by
  new code, resolve the contradiction explicitly (update the convention or
  fix the code) rather than leaving both to coexist.

## Examples

**Lesson entry format** (see `memory/lessons-learned.md` for the live file):

```markdown
### LL-0001 — Untrusted webhook payloads processed before signature check

- **Root Cause**: Webhook handler parsed and acted on payload fields before
  verifying the HMAC signature.
- **Why It Happened**: Signature verification was added after the initial
  handler and inserted in the wrong order during a later edit.
- **Solution**: Moved signature verification to the top of the handler;
  added a regression test that sends an invalid signature and asserts
  rejection before any parsing occurs.
- **Preventive Rule**: Any handler for external input must verify
  authenticity before deserializing payload contents. Added to
  `docs/standards/security.md`.
- **Similar Situations**: Any future webhook or callback handler; check
  `docs/standards/security.md` checklist before merging one.
```

## Common Mistakes

- Writing a lesson entry so generic it can't be searched for later (e.g.
  "fixed a bug in the API") — entries must name the specific mechanism.
- Letting `memory/roadmap.md` fall out of sync with actual priorities,
  causing new sessions to work on stale goals.
- Treating `memory/future-ideas.md` entries as commitments — they are not
  prioritized and should be clearly framed as "considered, not planned."
- Recording a lesson without adding the corresponding preventive rule to a
  standard or checklist — the lesson then has no teeth.

## Future Improvements

- Once `scripts/` automation matures, consider a lightweight indexing script
  that surfaces relevant `memory/` entries based on the files being changed
  in a given session (see Volume 05, extension points).
- Consider tagging lesson entries by domain (security, performance,
  concurrency, etc.) once the file grows large enough that a flat list is
  hard to scan.

## Related Documents

- `CLAUDE.md` §15 — Learning System (authoritative process)
- `docs/volumes/02-architecture.md` — where memory sits in the overall system
- `docs/decisions/` — full ADR content indexed by `memory/architecture-decisions.md`
- `memory/lessons-learned.md`, `memory/reusable-patterns.md`, `memory/roadmap.md`
