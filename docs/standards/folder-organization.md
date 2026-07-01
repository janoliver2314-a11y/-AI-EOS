# Standard: Folder Organization

## Purpose

Define where code, documentation, and configuration live so that "where
does this go?" always has a discoverable answer.

## Scope

Applies to the top-level repository structure and the internal organization
of each top-level directory as it grows.

## Goals

- Make the correct location for new content obvious without needing to ask.
- Keep each top-level directory internally consistent as it scales past a
  handful of files.

## Top-Level Layout

See `CLAUDE.md` §3.1 for the authoritative directory tree. In summary:

- `docs/` — all documentation, organized by type (volumes, standards,
  playbooks, diagrams, decisions, references).
- `memory/` — persistent, working project memory (not narrative docs).
- `prompts/` — reusable prompt templates.
- `templates/` — reusable document/code templates.
- `scripts/` — automation scripts.
- `src/` — application source code.
- `tests/` — automated tests, mirroring `src/` structure once it exists.
- `assets/` — static, non-code assets.

## Design Decisions

- **`docs/` is organized by document type, not by product area.** At this
  project's current size, a type-based split (standards vs. volumes vs.
  decisions) is more navigable than a feature-based one. Revisit if `docs/`
  grows large enough that type-based browsing becomes unwieldy.
- **`tests/` mirrors `src/` structure** once both exist, so a given source
  file's tests are always at the same relative path under `tests/`.
- **No `misc/` or `other/` directories.** If content doesn't fit an
  existing directory, that's a signal to either extend this standard
  deliberately (with an ADR if the change is structural) or reconsider the
  content's scope.

## Best Practices

- Before creating a new top-level directory, check whether the content
  actually belongs inside an existing one.
- Keep directory nesting shallow — prefer a well-named file over a deeply
  nested folder of near-empty files.
- When a directory grows past ~15–20 files, consider whether it needs
  subdirectories, and document the new sub-structure here.

## Examples

A new integration test suite: goes in `tests/integration/`, mirroring the
corresponding `src/` module path, not in a separate `integration-tests/`
top-level directory.

## Common Mistakes

- Creating a new top-level directory for something that fits inside
  `docs/`, `scripts/`, or `memory/`.
- Letting `src/` and `tests/` structures diverge, making it hard to find a
  given module's tests.
- Deeply nesting documentation (`docs/standards/backend/api/errors/handling.md`)
  when a flatter, well-named file would do.

## Future Improvements

- Once `src/` has real code, add a concrete example tree to this document
  showing `src/`/`tests/` mirroring in practice.

## Related Documents

- `CLAUDE.md` §3.1, §3.3
- `docs/standards/naming-conventions.md`
- `docs/volumes/02-architecture.md`
