# Standard: Dependency Management

## Purpose

Define the process for adding, updating, and removing external
dependencies (packages, GitHub Actions, MCP servers) so each one is a
deliberate, reviewed choice rather than an accumulated liability.

## Scope

Applies to any external dependency: language package managers, GitHub
Actions used in `.github/workflows/`, and MCP servers or other tool
integrations.

## Goals

- Prevent unreviewed or unnecessary dependencies from entering the
  codebase.
- Keep the dependency set auditable: for every dependency, it should be
  clear why it's there and who/what relies on it.
- Reduce supply-chain risk (`docs/volumes/07-security.md`).

## Rules

1. **Before adding a dependency, check if existing code/dependencies
   already solve the problem.** Check `memory/technology-choices.md` first.
2. **Justify the addition** — in the PR/commit description, state what
   problem it solves and why writing it in-house isn't the better choice
   for this project's scope.
3. **Vet the source**: maintenance activity, publisher reputation, and
   permissions/scope requested (especially for GitHub Actions and MCP
   servers, which can have broad access).
4. **Record the choice** in `memory/technology-choices.md` — what was
   chosen, what alternatives were considered, and why.
5. **Pin versions** deliberately (exact or range, per the project's
   versioning tooling once chosen) rather than always floating to latest.
6. **Remove unused dependencies** promptly rather than leaving them "in
   case they're needed later."

## Design Decisions

- **No dependency is added "just in case."** Every addition solves a
  concrete, current need (YAGNI, `CLAUDE.md` §4).
- **GitHub Actions and MCP servers are dependencies too** — they execute
  code with some level of trust and are subject to the same vetting as an
  npm package, not treated as inherently safe because they're
  "infrastructure."

## Best Practices

- Prefer widely-used, actively maintained packages over niche or abandoned
  ones, all else equal.
- When a dependency is only needed for one narrow use case, consider
  whether a small amount of in-house code is actually simpler to maintain
  long-term.
- Review a dependency's changelog before a major version upgrade, not just
  the diff in the lockfile.

## Examples

Adding a date-formatting library: check whether the runtime's standard
library already covers the need before adding a dependency; if not,
document the choice (`memory/technology-choices.md`) including why the
standard library was insufficient.

## Common Mistakes

- Adding a large dependency for a small piece of functionality that could
  be implemented directly.
- Installing a GitHub Action from an unfamiliar publisher without checking
  what repository permissions it requests.
- Leaving an unused dependency in the manifest after refactoring it away.
- Floating all dependencies to `latest` with no review process for
  updates, inviting unreviewed breaking or malicious changes.

## Future Improvements

- Add automated dependency vulnerability scanning to `.github/workflows/`
  once a package manifest exists.
- Document a periodic dependency review cadence once the dependency count
  is large enough to warrant it.

## Related Documents

- `docs/volumes/07-security.md`
- `memory/technology-choices.md`
- `docs/standards/versioning.md`
