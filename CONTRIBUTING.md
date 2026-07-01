# Contributing to AI-EOS

Thank you for considering a contribution to AI-EOS. This document describes
how to propose changes, the standards your change is expected to meet, and
how the review process works.

## Before You Start

1. Read `CLAUDE.md` in full — it is the operating manual for this repository
   and defines conventions that apply to every contribution.
2. Check `memory/roadmap.md` and open issues to see if your idea is already
   planned or in progress.
3. For anything beyond a small fix, open an issue describing the problem
   and your proposed approach before writing code — this avoids wasted work
   on approaches that won't be accepted.

## Development Workflow

1. Fork the repository (external contributors) or create a feature branch
   (maintainers): `feature/<short-description>` or `fix/<short-description>`.
2. Make your changes following the standards in `docs/standards/`.
3. Add or update tests for any behavior change (`docs/standards/testing.md`).
4. Update documentation affected by your change — including `CHANGELOG.md`
   under `[Unreleased]`.
5. If your change involves an architectural decision, add an ADR under
   `docs/decisions/` using `templates/adr-template.md`.
6. If your change fixes a bug, add an entry to `memory/lessons-learned.md`
   describing the root cause, per `CLAUDE.md` §15.
7. Open a pull request using `.github/PULL_REQUEST_TEMPLATE.md`.

## Commit Standards

Follow `CLAUDE.md` §11: a short imperative summary line, optionally followed
by a body explaining **why** the change was made. Keep commits scoped to one
logical change.

## Code Review

All pull requests are reviewed against the checklist in `CLAUDE.md` §12
before merge. Reviewers will ask for changes rather than rewrite your work;
please respond to review comments rather than opening a new PR unless asked.

## Documentation Contributions

Documentation is treated with the same rigor as code. New or updated
documents must follow the section skeleton in `docs/standards/documentation.md`
(Purpose, Scope, Goals, Design Decisions, Best Practices, Examples, Common
Mistakes, Future Improvements, Related Documents).

## Reporting Issues

- **Bugs**: use the bug report template in `.github/ISSUE_TEMPLATE/`.
  Include reproduction steps, expected vs. actual behavior, and environment
  details.
- **Feature requests**: use the feature request template. Explain the
  problem being solved, not just the desired solution.
- **Security issues**: do not open a public issue. See `docs/volumes/07-security.md`
  for the responsible disclosure process.

## Code of Conduct

All contributors are expected to follow `CODE_OF_CONDUCT.md`.

## Questions

Open a discussion or an issue tagged `question` if anything in this document
or in `CLAUDE.md` is unclear — ambiguity in these documents is itself a bug
worth fixing.
