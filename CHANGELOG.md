# Changelog

All notable changes to AI-EOS are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
once a first tagged release exists. Until then, changes are tracked under
`[Unreleased]`.

## [Unreleased]

### Changed

- `CLAUDE.md` §16 (Error Prevention Rules): added three rules on
  long-running shell commands — always enforce a self-terminating kill
  deadline, check host load (`uptime`) before assuming a slow command is a
  code bug, and kill the real process behind a wrapper (`npx`/`npm exec`),
  not just its `$!`. See `memory/lessons-learned.md#LL-0021`.
- `CLAUDE.md` §15 (Learning System): added an explicit rule requiring
  `memory/` to be consulted before implementing a feature or fixing a bug,
  and updated as part of the same change when new knowledge is gained —
  generalizing the prior lessons/patterns-only guidance to all of `memory/`.

### Added

- Initial repository bootstrap: directory structure, `CLAUDE.md` operating
  manual, ten-volume documentation library (`docs/volumes/`), engineering
  standards (`docs/standards/`), persistent project memory (`memory/`),
  document templates (`templates/`), automation extension points
  (`scripts/`, `.github/workflows/`), and contributor-facing documents
  (`README.md`, `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`, `LICENSE`).

### Notes

- This is a documentation-and-standards bootstrap. No application source
  code has shipped yet; `src/` and `tests/` are scaffolded with clear
  extension points per `docs/volumes/05-workflow-engine.md`.

<!--
When adding entries, use these categories as needed: Added, Changed,
Deprecated, Removed, Fixed, Security. Every entry should be understandable
without reading the linked PR/commit.
-->
