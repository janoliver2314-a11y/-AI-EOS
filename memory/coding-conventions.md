# Coding Conventions

Concrete, currently-enforced conventions. This file tracks what *is*
required in review, not aspirational style. For the reasoning behind these
conventions, see the linked standard.

## General (language-agnostic)

- File and directory naming: `docs/standards/naming-conventions.md`.
- Directory layout: `docs/standards/folder-organization.md`.
- Error handling contract: `docs/standards/error-handling.md`.
- Logging levels and redaction: `docs/standards/logging.md`.
- Commit message format: `CLAUDE.md` §11.

## Language/Framework-Specific

No primary application language has been chosen yet — `src/` is currently
unpopulated. Once a language/framework is selected (via ADR, see
`memory/architecture-decisions.md`), add a dedicated section here covering:

- Identifier casing (variables, functions, classes, constants)
- File-per-module conventions
- Formatting/linting tool and configuration
- Import/module organization rules

## Status

_Last reviewed: repository bootstrap (2026-07-01). No application code
exists yet; this file will grow as `src/` gains real conventions to
document._
