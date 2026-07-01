# Volume 01 — Foundation

## Purpose

Establish why AI-EOS exists, what problem it solves, and the philosophy that
every other document and decision in this repository builds on.

## Scope

This volume covers the project's mission, target users, guiding values, and
the high-level shape of the system. It does not cover implementation detail
(see Volume 02 — Architecture) or day-to-day standards (see Volume 06 —
Engineering Standards).

## Goals

- Give any new reader — human or AI — enough context to understand *why*
  AI-EOS is built the way it is, not just *what* it contains.
- Provide a stable reference point that other volumes can link back to
  instead of re-explaining foundational concepts.
- Make the project's values checkable: a proposed change should be
  evaluable against the principles in this document.

## The Problem

AI-assisted software engineering degrades over time without structure:

- Context resets between sessions, so decisions get re-litigated or
  silently contradicted.
- Standards drift because they live in someone's head instead of in a
  document the next session can read.
- The same bugs get reintroduced because lessons are never recorded
  anywhere durable.
- Documentation rots because it is treated as an afterthought rather than
  as a first-class deliverable.

AI-EOS exists to make an engineering codebase resilient to context loss —
whether that loss comes from a new AI session starting cold, a new engineer
joining, or six months passing between touches on a subsystem.

## Design Decisions

- **Documentation and memory are infrastructure, not overhead.** They are
  planned, reviewed, and maintained with the same rigor as code.
- **Modular volumes over one master document.** A single giant document
  becomes unmaintainable and un-linkable. Ten focused volumes, each with a
  clear scope, are easier to keep accurate and easier to navigate.
- **Standards are written down before they are enforced.** A rule that only
  exists as tribal knowledge cannot be followed reliably by a new
  contributor or a new AI session.
- **The system assumes zero-context readers.** Every document is written as
  if the reader has never seen this repository before, because for an AI
  session, that is frequently and literally true.

## Best Practices

- Treat every document as if it will be read by someone (or something)
  with no memory of this conversation.
- When a foundational assumption changes, update this volume rather than
  letting downstream documents silently diverge from it.
- Cross-link liberally. A reader should never hit a dead end where a
  concept is referenced but not explained or linked.
- Prefer updating an existing document over creating a parallel one that
  covers similar ground.

## Examples

A new AI session is asked to add a caching layer. Instead of guessing at
conventions, it:

1. Reads `CLAUDE.md` for global rules.
2. Reads `docs/volumes/02-architecture.md` to understand where a cache
   would fit.
3. Checks `memory/reusable-patterns.md` for an existing caching pattern.
4. Checks `memory/lessons-learned.md` for prior caching-related bugs.
5. Only then proposes an implementation, writing an ADR if the caching
   strategy is itself a significant decision.

## Common Mistakes

- Treating documentation as something to write *after* the "real work" is
  done — this reliably means it never gets written accurately.
- Writing documents that describe *what* code does instead of *why* a
  decision was made — the former rots as code changes; the latter doesn't.
- Creating a new document instead of updating an existing one, resulting in
  two half-accurate sources of truth.
- Skipping the memory system because "the fix was small" — small fixes with
  no recorded root cause are exactly the ones that reappear later.

## Future Improvements

- Add a lightweight onboarding checklist that walks a first-time reader
  (human or AI) through Volumes 01–03 in order.
- Consider tooling that flags documents which haven't been touched despite
  the code they describe changing (staleness detection), once `scripts/`
  automation is built out per Volume 05.

## Related Documents

- `CLAUDE.md` — operating manual and authoritative rule set
- `docs/volumes/02-architecture.md` — system architecture
- `docs/volumes/03-memory-system.md` — how project memory works
- `docs/volumes/06-engineering-standards.md` — standards overview
- `docs/volumes/10-master-reference.md` — index of all volumes
