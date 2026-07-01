# Standard: Testing

## Purpose

Define what must be tested, how tests should be structured, and what
"adequately tested" means for a change to be considered Done.

## Scope

Applies to all behavior changes in `src/`. Documentation-only or pure
configuration changes are exempt unless they affect runtime behavior.

## Goals

- Ensure every behavior change is protected against regression.
- Keep tests fast, deterministic, and meaningful rather than a coverage
  metric to satisfy.
- Make test intent obvious from its name and structure alone.

## Test Types

| Type | Purpose | Location |
|---|---|---|
| Unit | A single function/module in isolation | `tests/unit/`, mirroring `src/` |
| Integration | Multiple components working together | `tests/integration/` |
| End-to-end | Full user-facing flow | `tests/e2e/` |
| Regression | Reproduces a specific fixed bug | Co-located with the relevant unit/integration test, referencing the `memory/lessons-learned.md` entry |

## Rules

1. **New behavior ships with a test that fails without the change** and
   passes with it — this is the definition of "tested," not merely "a test
   exists somewhere."
2. **Bug fixes ship with a regression test** that reproduces the original
   failure, per `CLAUDE.md` §9.
3. **Test names describe behavior**, not implementation: `rejects a request
   with an invalid signature`, not `test1` or `testHandler`.
4. **No skipped or pending tests without a linked tracking issue** and an
   inline comment explaining why.
5. **Tests are deterministic** — no reliance on real wall-clock time,
   network calls to live services, or unseeded randomness in unit tests.
6. **Flaky tests are fixed or removed**, never silently ignored/retried
   into passing.

## Design Decisions

- **Coverage percentage is not the goal; behavior coverage is.** A high
  percentage with untested edge cases is worse than a lower percentage
  where every documented behavior (including failure modes) is verified.
- **Regression tests explicitly reference their lesson entry** so the link
  between "bug that happened" and "test that prevents recurrence" stays
  visible.

## Best Practices

- Write the failing test before the fix when practical (see
  `docs/playbooks/` for a bug-fix procedure) — this proves the test would
  have caught the bug.
- Test the documented error contract (`docs/standards/error-handling.md`),
  not just the happy path.
- Keep test setup minimal and explicit; avoid shared mutable fixtures that
  make tests order-dependent.

## Examples

```text
describe("webhook signature verification", () => {
  it("rejects a payload with an invalid signature", () => { ... });
  it("rejects a payload with a missing signature header", () => { ... });
  it("accepts a payload with a valid signature", () => { ... });
});
```

## Common Mistakes

- Writing a test that only exercises the happy path, leaving documented
  error conditions unverified.
- Naming tests after the function under test rather than the behavior
  being verified, making failures hard to interpret at a glance.
- Adding `skip`/`todo` tests that are never revisited.
- Asserting on incidental implementation detail (internal call counts,
  private state) instead of observable behavior.

## Future Improvements

- Choose a test runner/framework once a language is selected for `src/`
  and document project-specific conventions (mocking approach, fixture
  patterns) here.
- Add CI enforcement (`.github/workflows/`) once a test suite exists, per
  `docs/volumes/05-workflow-engine.md`.

## Related Documents

- `CLAUDE.md` §9, §17
- `docs/standards/error-handling.md`
- `memory/lessons-learned.md`
