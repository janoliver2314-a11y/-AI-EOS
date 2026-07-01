# Standard: Error Handling

## Purpose

Define how errors are represented, propagated, and surfaced so failures are
diagnosable without reproduction and never fail silently.

## Scope

Applies to all application code under `src/` and to automation scripts
under `scripts/`. Documentation-only work is out of scope.

## Goals

- Make every failure mode of a public function/module explicit and
  intentional, not accidental.
- Ensure failures carry enough context to diagnose without needing to
  reproduce them.
- Prevent silent failure — a caught error that is neither handled nor
  reported is a bug.

## Rules

1. **Every public function/module defines its error contract** — what it
   throws, returns, or rejects with on failure, and under what conditions.
2. **Errors carry context**, not just a message: what operation was being
   attempted and with what relevant (non-sensitive) inputs.
3. **Catch only what you can handle.** A catch block that can't meaningfully
   recover should log/re-throw with context, not swallow the error.
4. **No empty catch blocks.** If an error is genuinely safe to ignore, say
   so explicitly in a comment explaining why.
5. **User-facing error messages are distinct from internal error detail.**
   Internal detail (stack traces, internal identifiers) is logged, not
   shown to end users; user-facing messages are actionable and free of
   internal implementation detail.
6. **Validate at boundaries, trust internally.** Once input has been
   validated at a system boundary, internal code should not re-validate
   defensively against conditions that can't occur.

## Design Decisions

- **No silent fallbacks** (`CLAUDE.md` §4) — a fallback value returned after
  a failure must be an explicit, documented design choice, not a way to
  avoid deciding how to handle the error.
- **Errors are part of the interface**, documented alongside the function's
  normal behavior, not as an afterthought.

## Best Practices

- Name custom error types for the failure condition they represent
  (`ValidationError`, `RateLimitExceededError`), not generically
  (`AppError`, `CustomError`) unless used as a common base type.
- Include enough context in an error to answer "what was being attempted
  and why did it fail" without needing to add debug logging after the
  fact.
- Write a test for each documented failure mode, not just the happy path
  (see `docs/standards/testing.md`).

## Examples

```text
// Good: specific, contextual, and part of the documented contract
throw new ValidationError(
  `Invalid webhook signature for event ${eventId}`
);

// Bad: swallows the failure, caller has no idea anything went wrong
try {
  processWebhook(payload);
} catch (e) {
  // ignore
}
```

## Common Mistakes

- Catching an error broadly (e.g. catching all exceptions around a large
  block) and only handling one specific case, silently swallowing the
  rest.
- Returning `null`/`undefined`/a default value on failure without
  documenting that this is the intended failure signal.
- Logging an error and then continuing as if it succeeded.
- Exposing internal error detail (stack traces, database errors) directly
  to end users.

## Future Improvements

- Once a language/framework is chosen for `src/`, add concrete syntax
  examples and any framework-specific error-handling conventions.

## Related Documents

- `docs/standards/logging.md`
- `docs/standards/testing.md`
- `docs/standards/reliability.md`
- `CLAUDE.md` §4, §5
