# Standard: Versioning

## Purpose

Define how AI-EOS versions its releases and handles breaking changes, so
consumers (including future contributors) can reason about compatibility.

## Scope

Applies once the project has tagged releases. Applies to any future
published package, API, or CLI, and to the documentation/standards system
itself where a breaking convention change occurs.

## Goals

- Make compatibility expectations explicit and predictable.
- Ensure breaking changes are never silent.

## Rules

1. **Semantic Versioning (`MAJOR.MINOR.PATCH`)** once a first release is
   tagged:
   - `MAJOR` — breaking changes to a public interface/contract.
   - `MINOR` — backwards-compatible functionality additions.
   - `PATCH` — backwards-compatible bug fixes.
2. **Every release is recorded in `CHANGELOG.md`** before the tag is
   created, under the version heading, using Keep a Changelog categories
   (Added, Changed, Deprecated, Removed, Fixed, Security).
3. **Breaking changes require an ADR** (`docs/decisions/`) explaining the
   change and migration path, in addition to the CHANGELOG entry.
4. **Pre-1.0, breaking changes may occur in `MINOR` releases**, but must
   still be documented; this project should move to `1.0.0` once its core
   interfaces are considered stable.

## Design Decisions

- **Documentation conventions are versioned informally via CHANGELOG, not
  SemVer**, since `docs/` and `memory/` are living documents rather than a
  published interface. A structural convention change (e.g. renaming the
  document skeleton sections) is still ADR-worthy per `CLAUDE.md` §14.
- **The repository stays pre-1.0 until there is a concrete, stable public
  interface** (a library API, CLI, or service contract) to version against.

## Best Practices

- Tag releases only from the default branch, per
  `docs/volumes/08-deployment.md`.
- Write CHANGELOG entries a reader can understand without reading the
  underlying commit/PR.
- When deprecating something, mark it `Deprecated` in the CHANGELOG for at
  least one release cycle before removal, with a stated removal version.

## Examples

```markdown
## [1.2.0] - 2026-08-01
### Added
- Rate limiting for the webhook handler.
### Fixed
- Webhook signature check now runs before payload parsing (LL-0001).
```

## Common Mistakes

- Making a breaking change in a `PATCH` or `MINOR` release post-1.0.
- Tagging a release without a corresponding CHANGELOG entry.
- Removing a deprecated feature without having announced the deprecation
  in a prior release.

## Future Improvements

- Automate CHANGELOG-to-release-notes generation once `.github/workflows/`
  has a release pipeline (`docs/volumes/08-deployment.md`).

## Related Documents

- `CHANGELOG.md`
- `docs/volumes/08-deployment.md`
- `docs/decisions/`
