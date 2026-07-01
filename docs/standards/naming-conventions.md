# Standard: Naming Conventions

## Purpose

Ensure that names — of files, directories, branches, commits, and code
identifiers — are predictable enough that anyone (human or AI) can guess
where something lives or what something does before opening it.

## Scope

Applies to all files and identifiers in this repository. Language-specific
identifier casing rules apply once a language is chosen for `src/` (tracked
in `memory/technology-choices.md`); this document defines the
language-agnostic rules that apply everywhere now.

## Goals

- Eliminate guesswork about where a new file belongs or what an existing
  one contains, from its name alone.
- Keep names stable over time so links (in docs, code, and git history)
  don't rot.

## Rules

| Item | Convention | Example |
|---|---|---|
| Markdown documents | `kebab-case.md` | `error-handling.md` |
| Volumes | `NN-topic-name.md` (2-digit prefix) | `06-engineering-standards.md` |
| ADRs | `NNNN-short-title.md` (4-digit prefix, never reused) | `0007-choose-database.md` |
| Directories | `kebab-case` or `lowercase` single word | `docs/`, `docs/standards/` |
| Git branches | `feature/<short-description>`, `fix/<short-description>`, `docs/<short-description>` | `feature/rate-limiting` |
| Commits | Imperative summary line, ≤ 72 chars | `Add retry policy for MCP tool calls` |
| Environment variables | `SCREAMING_SNAKE_CASE` | `DATABASE_URL` |

## Design Decisions

- **Numeric prefixes are used only where stable ordering matters** (volumes,
  ADRs). Everything else uses descriptive names, since imposing numeric
  order elsewhere adds churn without benefit.
- **ADR numbers are never reused or renumbered**, even if an ADR is
  superseded — the number is a permanent identifier, not a priority order.

## Best Practices

- Name a file for what it contains, not for the ticket or task that
  produced it — task references belong in commit messages, not filenames.
- If you can't name a new document in one clear phrase, it may be trying to
  cover more than one topic — consider splitting it.
- Keep branch names short enough to read in a PR list at a glance.

## Examples

Good: `docs/standards/dependency-management.md`, `feature/webhook-signature-check`.

Avoid: `docs/standards/misc.md` (unclear scope), `fix-bug-2` (no context),
`Docs/Standards/DependencyManagement.MD` (inconsistent casing).

## Common Mistakes

- Mixing casing styles within the same directory.
- Using ticket/issue numbers as the primary filename component instead of a
  descriptive name.
- Renumbering ADRs when one is superseded — supersession is recorded via a
  link between the two documents, not a renumber.

## Future Improvements

- Add language-specific identifier conventions (variables, functions,
  classes) once a primary language is chosen for `src/`.

## Related Documents

- `docs/standards/folder-organization.md`
- `docs/standards/versioning.md`
- `CLAUDE.md` §3.2
