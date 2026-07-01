# Tests

Automated test suites for AI-EOS, mirroring `src/`'s structure once it
exists, per `docs/standards/folder-organization.md`.

## Planned Structure

```
tests/
├── unit/          # mirrors src/, one function/module at a time
├── integration/   # multiple components together
└── e2e/           # full user-facing flows
```

See `docs/standards/testing.md` for what must be tested and how tests
should be named and structured.

## Related Documents

- `docs/standards/testing.md`
- `docs/standards/folder-organization.md`
- `memory/lessons-learned.md` — regression tests reference their lesson entry
