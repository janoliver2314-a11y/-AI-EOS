# Scripts

Automation scripts for AI-EOS. Per `docs/volumes/05-workflow-engine.md`,
these are extension points: this directory currently has no scripts,
because there is no application code yet for automation to operate on. Each
row below is a designed seam, not an implementation.

## Planned Extension Points

| Script (future) | Purpose | Attaches at workflow step |
|---|---|---|
| `docs-index.{sh,js}` | Regenerate cross-links in `docs/README.md` / Volume 10 | Record (step 6) |
| `lint.{sh,js}` | Run formatting/lint checks locally, matching CI | Test (step 4) |
| `test.{sh,js}` | Run the test suite locally, matching CI | Test (step 4) |
| `check-standards.{sh,js}` | Verify new docs follow the skeleton in `docs/standards/documentation.md` | Review (step 5) |

## Design Decisions

- No script is added here until there is a concrete, current need for it —
  adding automation speculatively produces untested, unmaintained tooling
  (`docs/volumes/05-workflow-engine.md` Common Mistakes).
- Every script added here must be documented in this README with its
  purpose, inputs, outputs, and failure mode before or alongside being
  merged.

## Related Documents

- `docs/volumes/05-workflow-engine.md`
- `.github/workflows/README.md`
