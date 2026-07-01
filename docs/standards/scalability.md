# Standard: Scalability

## Purpose

Define how AI-EOS designs for growth — in data volume, usage, and
codebase size — without paying for complexity the project doesn't need
yet.

## Scope

Applies to architectural and design decisions in `src/`, and to the
documentation/memory system's own ability to scale as it grows.

## Goals

- Make deliberate, documented tradeoffs between current simplicity and
  future growth, rather than defaulting to either extreme.
- Ensure the documentation and memory system itself scales gracefully as
  content grows (this system practices what it documents).

## Design Decisions

- **Design for the next order of magnitude, not the eventual maximum.**
  Building for a scale far beyond current or near-term needs is premature
  complexity (`CLAUDE.md` §4); building with no room to grow at all creates
  a rewrite trap. Aim for "this will comfortably handle 10x current load"
  as a default heuristic, adjusted per context.
- **Scalability decisions with real tradeoffs are ADR-worthy**
  (`CLAUDE.md` §14) — e.g. choosing a data storage approach, a caching
  strategy, or a service decomposition boundary.
- **The documentation system scales by splitting, not by growing single
  files.** This is why `docs/` is organized into many small, focused
  documents (Volume 01, Volume 06) instead of a few large ones — the same
  principle should guide `src/` module boundaries.

## Best Practices

- Identify the actual growth dimension that matters (requests/sec, data
  volume, number of documents, number of contributors) before designing
  around it.
- Prefer horizontal decomposition (more small, focused units) over vertical
  complexity (fewer units doing more) as the default scaling strategy,
  consistent with `docs/standards/folder-organization.md`.
- Revisit scalability assumptions when a directory, module, or dataset
  crosses a size threshold that makes the current approach awkward (see
  `docs/standards/folder-organization.md`'s ~15–20 file guideline as one
  example).

## Examples

`docs/standards/` scaling: rather than one large `standards.md` growing
indefinitely, each topic is its own file. When a topic itself grows too
large (e.g. `security.md` covering ten distinct subtopics), split it
further with cross-links, following the same pattern.

## Common Mistakes

- Building a distributed, horizontally-scaled architecture for a system
  with a handful of users and no evidence it will grow beyond that soon.
- Ignoring scale entirely and hard-coding assumptions (e.g. "there will
  only ever be one of these") that are expensive to unwind later.
- Letting a single document or module grow indefinitely instead of
  splitting it once it becomes hard to navigate.

## Future Improvements

- Once `src/` has a real data layer, document concrete scaling patterns
  (pagination, batching, sharding considerations) specific to the chosen
  technology.

## Related Documents

- `docs/standards/performance.md`
- `docs/standards/reliability.md`
- `docs/volumes/02-architecture.md`
