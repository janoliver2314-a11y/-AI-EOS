# Standard: Reliability

## Purpose

Define how AI-EOS designs for failure — of dependencies, networks, and
its own components — so degradation is graceful and recovery is
predictable.

## Scope

Applies to any code that depends on an external system (network calls,
file I/O, third-party APIs, MCP servers) and to automation in `scripts/`
and `.github/workflows/`.

## Goals

- Ensure transient failures don't become hard failures where avoidable.
- Make failure modes predictable and recoverable, with a clear owner for
  responding to them.
- Avoid reliability mechanisms (retries, circuit breakers) that themselves
  introduce new failure modes through misuse.

## Rules

1. **Distinguish transient from permanent failures.** Retry logic applies
   only to failures plausibly caused by transient conditions (network
   blips, rate limits); permanent failures (invalid input, auth failure)
   should fail immediately.
2. **Retries are bounded and backed off.** Unbounded retries or tight
   retry loops turn a transient issue into a self-inflicted denial of
   service.
3. **External calls have timeouts.** No call to an external system should
   be allowed to hang indefinitely.
4. **Degrade gracefully where possible**, but never silently — a degraded
   mode must be observable (logged/reported), per
   `docs/standards/logging.md`.
5. **Idempotency matters for anything retried** — a retried operation must
   not produce duplicate side effects.

## Design Decisions

- **No silent fallbacks** (`CLAUDE.md` §4) — reliability mechanisms
  (retries, fallbacks, circuit breakers) must surface that they activated,
  not just quietly paper over the underlying failure.
- **Reliability patterns are added when there's a real dependency to
  protect against**, not speculatively before any external dependency
  exists.

## Best Practices

- Use exponential backoff with jitter for retries against shared external
  services, to avoid synchronized retry storms.
- Document the expected failure modes of any external dependency (rate
  limits, downtime patterns) alongside the code that calls it.
- Test failure paths explicitly (simulated timeouts, error responses), not
  just the success path (`docs/standards/testing.md`).

## Examples

```text
// Retry only on transient network/5xx errors, bounded, with backoff
async function callWithRetry(fn, { maxAttempts = 3 } = {}) {
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (err) {
      if (!isTransient(err) || attempt === maxAttempts) throw err;
      await sleep(backoffMs(attempt));
    }
  }
}
```

## Common Mistakes

- Retrying non-idempotent operations (e.g. "create" calls) without
  deduplication, risking duplicate side effects.
- Retrying permanent failures (e.g. 400 Bad Request) as if they were
  transient, wasting time and obscuring the real problem.
- Adding a circuit breaker or retry layer before there's a real external
  dependency that needs it.
- Silent degraded modes that leave no trace they activated.

## Future Improvements

- Once `src/` integrates real external dependencies, document their
  specific failure modes and the chosen resilience pattern for each.

## Related Documents

- `docs/standards/error-handling.md`
- `docs/standards/logging.md`
- `docs/standards/performance.md`
