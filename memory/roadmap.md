# Roadmap

Near/mid-term plan, in priority order. This is the tactical counterpart to
`docs/volumes/09-business-strategy.md`'s longer-term vision — expect this
file to change often.

## Now

- [x] Bootstrap repository structure, `CLAUDE.md`, documentation library,
      standards, memory system, and automation extension points.
- [ ] Decide on the first concrete application/use case to build under
      `src/`, recorded via ADR once chosen.

## Next

- [ ] Choose a primary language/framework for `src/` (ADR).
- [ ] Stand up a minimal CI workflow (lint + test) once there is code to
      check (`docs/volumes/05-workflow-engine.md` extension point).
- [ ] Build the first real automation extension: a documentation index
      generator keeping `docs/README.md` / Volume 10 cross-links accurate.

## Later

- [ ] Deployment pipeline once a first deployable artifact exists
      (`docs/volumes/08-deployment.md`).
- [ ] Dependency vulnerability scanning in CI.
- [ ] Evaluate adoption of AI-EOS's structure in a second repository to
      validate its portability (`docs/volumes/09-business-strategy.md`).

## Status

_Last updated: repository bootstrap (2026-07-01)._
