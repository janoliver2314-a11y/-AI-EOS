# Standard: Documentation

## Purpose

Define the required structure and writing discipline for documentation in
this repository, so every document is complete, comparable, and useful to a
zero-context reader.

## Scope

Applies to `docs/volumes/`, `docs/standards/`, `docs/playbooks/`, and any
other substantive document. Short reference files (README indexes,
templates) follow a lighter version of this structure where the full
skeleton would be redundant.

## Goals

- Guarantee every substantive document answers the same core questions, so
  readers know what to expect and don't hit gaps.
- Keep documentation accurate over time by making it easy to see what's
  stale.

## The Document Skeleton

Every substantive document includes these sections, in this order:

1. **Purpose** — why this document exists.
2. **Scope** — what it covers and, explicitly, what it does not.
3. **Goals** — what a reader/user of this document should be able to do
   after reading it.
4. **Design Decisions** — the reasoning behind non-obvious choices,
   including alternatives considered where relevant.
5. **Best Practices** — concrete, actionable guidance.
6. **Examples** — at least one concrete example of correct application.
7. **Common Mistakes** — specific, real failure patterns to avoid.
8. **Future Improvements** — known gaps or deferred enhancements.
9. **Related Documents** — explicit links; never assume discoverability.

## Design Decisions

- **The skeleton is fixed** so documents are comparable and complete;
  deviating from it (skipping a section) should be rare and deliberate,
  not a shortcut.
- **"Common Mistakes" must be specific.** A generic mistake ("be careful")
  provides no value; a specific one ("catching a broad exception type and
  silently continuing") is checkable.
- **No placeholder sections.** A document with "Future Improvements: TBD"
  should either write a real (even if small) list or omit content — an
  empty stub signals the document isn't finished.

## Best Practices

- Write for a reader with zero context on this specific conversation or
  task — assume they've read `CLAUDE.md` and nothing else.
- Prefer updating an existing document to creating a new one covering
  similar ground (`CLAUDE.md` §6).
- Link related documents by relative path so links survive repository
  moves/renames within the same structure.
- Keep examples realistic and runnable/checkable where possible, not
  abstract pseudocode when concrete code would do.

## Examples

This document is itself an example of the skeleton applied to a standard.
See any file in `docs/volumes/` for the skeleton applied to narrative
documentation.

## Common Mistakes

- Writing "Common Mistakes" as restated best practices in negative form
  ("don't skip best practices") instead of concrete failure patterns.
- Omitting "Related Documents," leaving a reader without a path to more
  detail.
- Letting a document's content drift from reality without updating it —
  treat stale documentation as a bug.
- Creating a "v2" document instead of updating the original in place.

## Future Improvements

- Once `scripts/` automation exists, consider a linter that checks new
  Markdown files under `docs/` for the required section headings.

## Related Documents

- `CLAUDE.md` §6
- `docs/volumes/06-engineering-standards.md`
- `docs/volumes/10-master-reference.md`
