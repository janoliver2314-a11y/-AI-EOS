# GitHub Actions Workflows

No workflows are implemented yet — this is a designed extension point per
`docs/volumes/05-workflow-engine.md`. Building CI before there is code to
lint/test/build would produce untested, speculative automation
(`docs/volumes/05-workflow-engine.md` Common Mistakes).

## Planned Workflows

| Workflow (future) | Trigger | Purpose |
|---|---|---|
| `ci.yml` | Pull request | Lint + run test suite (`docs/standards/testing.md`) |
| `security-scan.yml` | Pull request, schedule | Dependency vulnerability scanning (`docs/standards/dependency-management.md`) |
| `release.yml` | Version tag push | Build and publish a tagged release (`docs/volumes/08-deployment.md`) |

## Design Decisions

- Each workflow added here must be documented in this README (trigger,
  purpose, required secrets/permissions) before or alongside being merged,
  per `docs/volumes/05-workflow-engine.md` Best Practices.
- Workflow credentials follow least-privilege
  (`docs/volumes/07-security.md`); document required repository secrets
  here without ever committing their values.

## Related Documents

- `docs/volumes/05-workflow-engine.md`
- `docs/volumes/08-deployment.md`
- `docs/standards/dependency-management.md`
