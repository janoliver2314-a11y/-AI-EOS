# Standard: Logging

## Purpose

Define what gets logged, at what level, and how, so logs are useful for
diagnosis without becoming a liability (noise, cost, or a source of
leaked sensitive data).

## Scope

Applies to all application code and automation under `src/` and `scripts/`.

## Goals

- Make production issues diagnosable from logs alone in the common case.
- Prevent sensitive data (secrets, credentials, PII) from ever reaching log
  output.
- Keep log volume proportional to its diagnostic value.

## Log Levels

| Level | Use for |
|---|---|
| `error` | A failure that needs attention; an operation did not complete as intended |
| `warn` | An unexpected but handled condition; degraded operation |
| `info` | Significant, expected lifecycle events (startup, request completion, job finished) |
| `debug` | Detailed diagnostic information, off by default in production |

## Rules

1. **Never log secrets, credentials, tokens, or full PII.** Redact or omit
   sensitive fields before logging; when in doubt, don't log the field.
2. **Every `error`-level log includes context**: what operation, what
   (non-sensitive) inputs, and the underlying error.
3. **`info` level should be readable as a timeline** of what the system did,
   without being so verbose it drowns out signal.
4. **Structured logging over string concatenation** — log key-value data
   that can be filtered/queried, not just human-readable sentences, once a
   logging library is chosen.
5. **No logging in tight loops** without explicit rate limiting or
   aggregation — this is both a performance and a noise concern.

## Design Decisions

- **Redaction is opt-out, not opt-in, for known-sensitive field names**
  (e.g. `password`, `token`, `secret`, `authorization`) once a logging
  utility is implemented — defaulting to safe behavior prevents a single
  missed case from leaking a credential.
- **Log level is a contract, not a suggestion** — using `error` for
  expected/handled conditions causes alert fatigue and trains people to
  ignore real errors.

## Best Practices

- Log at the point where you have the most context about what was being
  attempted, not just where the error surfaces.
- Include a correlation/request identifier in logs for any operation that
  spans multiple steps, once such an identifier exists in the system.
- Review log output for a new feature before merging — does it tell a
  useful story of what happened, or is it noise?

## Examples

```text
// Good
logger.error("Failed to process webhook", { eventId, cause: err.message });

// Bad — leaks the payload, which may contain sensitive fields
logger.error("Failed to process webhook", { payload });
```

## Common Mistakes

- Logging entire request/response objects "for debugging" and leaving it
  in production code.
- Using `console.log`/print statements left over from debugging instead of
  the project's logging utility.
- Logging the same error at multiple layers as it propagates, producing
  duplicate noise for a single failure.
- Treating `debug`-level logs as safe to leave verbose in production by
  default — verify the default level before shipping.

## Future Improvements

- Choose and document a logging library/utility once `src/` has real code,
  and add a redaction helper as a shared utility rather than per-call-site
  discipline alone.

## Related Documents

- `docs/standards/error-handling.md`
- `docs/volumes/07-security.md`
- `docs/standards/security.md`
