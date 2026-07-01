# Volume 08 — Deployment

## Purpose

Define the principles AI-EOS will follow for releasing and deploying
software, so that when a first deployable artifact exists, the process is
already agreed rather than improvised.

## Scope

Covers release/versioning philosophy, environment strategy, and the shape
of the deployment pipeline as an extension point. Does not describe a
concrete deployed system yet, since none exists — this volume is the
contract a future deployment implementation must satisfy.

## Goals

- Ensure the first real deployment follows a deliberate process rather than
  an ad hoc one improvised under time pressure.
- Keep environments and release steps predictable and reversible.
- Define where deployment automation attaches (see Volume 05) without
  building it prematurely.

## Environment Strategy

AI-EOS assumes, at minimum, three environment tiers once an application
exists:

| Environment | Purpose | Deploys from |
|---|---|---|
| **Local** | Development and manual testing | Any branch, developer machine |
| **Staging** | Pre-production validation | Default branch, automatically or on demand |
| **Production** | Live system | Tagged releases only |

## Design Decisions

- **Releases are tagged, not branch-based.** Production deploys from an
  explicit version tag (see `docs/standards/versioning.md`), not from
  whatever the default branch currently contains — this keeps "what's in
  production" always answerable from git history alone.
- **Deployment is automatable but never silent.** Whatever pipeline
  eventually deploys to production must produce a durable, reviewable
  record (a CHANGELOG entry, a release note, or both) — no deploy should
  be untraceable after the fact.
- **Rollback is a first-class requirement, designed in from the start.**
  Any deployment mechanism built must have a documented rollback procedure
  before it is used for production traffic.
- **No deployment automation is implemented until there is something to
  deploy.** Building a deployment pipeline against a nonexistent
  application produces untested, unmaintainable automation.

## Best Practices

- Version every release per `docs/standards/versioning.md` (Semantic
  Versioning) and record it in `CHANGELOG.md` before tagging.
- Keep staging as close to production configuration as practical, so
  staging validation is meaningful.
- Any manual deployment step that is performed more than once should be
  scripted and documented in `scripts/`.
- Treat deployment credentials with the same least-privilege discipline as
  any other credential (`docs/volumes/07-security.md`).

## Examples

A future first release, once `src/` has a deployable application:

1. Merge changes to the default branch following the standard workflow
   (Volume 05).
2. Update `CHANGELOG.md` under a new version heading.
3. Tag the release (`vX.Y.Z`) per `docs/standards/versioning.md`.
4. CI (`.github/workflows/`) builds and deploys the tag to staging
   automatically; production deploy requires explicit approval.
5. Record the deployment (date, version, notable changes) in a release
   note or `CHANGELOG.md` entry.

## Common Mistakes

- Deploying directly from an unreviewed branch "just this once" — this is
  precisely how untraceable production incidents happen.
- Building a complex multi-environment pipeline before there is a single
  working deployable artifact to justify it.
- Treating staging as optional under deadline pressure, which removes the
  only pre-production validation step.
- Skipping a rollback plan because the initial deploy "should be fine."

## Future Improvements

- Once a concrete application and hosting target are chosen (via ADR), add
  a hosting-specific deployment guide under `docs/playbooks/`.
- Add deployment automation to `.github/workflows/` following the
  extension-point contract in `docs/volumes/05-workflow-engine.md`.

## Related Documents

- `docs/volumes/05-workflow-engine.md` — where deployment automation attaches
- `docs/standards/versioning.md` — release versioning rules
- `docs/volumes/07-security.md` — credential handling for deployment automation
- `CHANGELOG.md` — release history
