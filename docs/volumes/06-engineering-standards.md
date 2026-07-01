# Volume 06 — Engineering Standards

## Purpose

Provide the narrative overview of AI-EOS's engineering standards and explain
how the individual standard documents in `docs/standards/` relate to each
other and to `CLAUDE.md`.

## Scope

This volume is an index and rationale layer. The normative, checkable rules
themselves live in `docs/standards/*.md`, one topic per file. This volume
does not restate those rules in full — it explains why the set is organized
this way and how to use it.

## Goals

- Make the standards set navigable: a reader should be able to find the
  relevant standard for any given change in under a minute.
- Explain the relationship between `CLAUDE.md` (always-applicable summary
  rules) and `docs/standards/` (detailed, topic-specific rules).
- Keep standards enforceable by keeping them concrete and example-driven,
  not aspirational prose.

## The Standards Set

| Document | Covers |
|---|---|
| `docs/standards/naming-conventions.md` | Identifiers, files, branches, commits |
| `docs/standards/folder-organization.md` | Where code and docs live and why |
| `docs/standards/error-handling.md` | Error contracts, propagation, user-facing messages |
| `docs/standards/logging.md` | What/when/how to log, levels, redaction |
| `docs/standards/testing.md` | Test types, coverage expectations, naming |
| `docs/standards/documentation.md` | The document skeleton and writing rules |
| `docs/standards/dependency-management.md` | Adding, updating, and removing dependencies |
| `docs/standards/versioning.md` | SemVer usage, breaking-change policy |
| `docs/standards/performance.md` | Performance expectations and measurement discipline |
| `docs/standards/scalability.md` | Designing for growth without premature complexity |
| `docs/standards/reliability.md` | Failure handling, retries, graceful degradation |
| `docs/standards/security.md` | Threat handling, secrets, dependency vetting |

## Design Decisions

- **One topic per file.** A single "coding standards" mega-document becomes
  stale and hard to review incrementally; per-topic files can be updated
  independently and linked precisely.
- **`CLAUDE.md` holds summaries; `docs/standards/` holds the detail.** This
  avoids the summary drifting out of sync with detail by keeping the
  summary intentionally short and pointing to one canonical detailed
  source per topic.
- **Standards are descriptive of enforced practice, not aspirational.** A
  standard describes what is actually required in review, not a "someday"
  ideal — aspirational goals belong in `memory/future-ideas.md` or
  `memory/roadmap.md` until they're ready to be enforced.

## Best Practices

- Before writing code in a new area, skim the relevant standard(s) rather
  than relying on general knowledge — AI-EOS's specifics (e.g. its error
  contract shape) may differ from generic best practice.
- When a standard needs to change, update the file in place and note the
  change in `CHANGELOG.md`; do not create a competing document.
- If two standards appear to conflict, treat that as a bug in the standards
  set — resolve it explicitly rather than picking one silently.

## Examples

A contributor adding a new npm dependency:

1. Reads `docs/standards/dependency-management.md` for the vetting criteria.
2. Checks `memory/technology-choices.md` for whether an existing dependency
   already covers the need.
3. If adding it, documents the choice in `memory/technology-choices.md`.

## Common Mistakes

- Writing a new standard as a wall of prose instead of concrete,
  checkable rules with examples of compliant and non-compliant code.
- Duplicating a rule from `CLAUDE.md` into a standards file verbatim
  instead of expanding on it — duplication is how the two drift apart.
- Adding a standard nobody enforces in review, which trains contributors to
  ignore the standards set generally.

## Future Improvements

- As `src/` gains real code, add language-specific addenda to
  `docs/standards/naming-conventions.md` rather than a full new document.
- Consider lightweight lint rules that encode the most mechanical standards
  (naming, formatting) once a toolchain is chosen (see
  `docs/decisions/`).

## Related Documents

- `CLAUDE.md` §5, §6, §12 — coding standards summary, documentation
  standards summary, review checklist
- `docs/standards/` — the full standards set
- `memory/coding-conventions.md` — current, concrete conventions in force
