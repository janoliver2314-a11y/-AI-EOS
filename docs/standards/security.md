# Standard: Security

## Purpose

Provide concrete, checkable security rules that apply to every change,
complementing the reasoning and process described in
`docs/volumes/07-security.md`.

## Scope

Applies to all code, configuration, and automation in this repository.

## Goals

- Make secure-by-default behavior the easy, obvious path.
- Give reviewers a concrete checklist for security-sensitive changes.

## Rules

1. **No secrets in git** — not in commits, history, logs, or example
   files. Use `.env.example` with placeholder values; document required
   variables there.
2. **Validate all external input at the boundary** (user input, webhook
   payloads, third-party API responses) before acting on it. Verify
   authenticity (e.g. signatures) before parsing content, not after.
3. **Least privilege for every credential** — scope tokens, service
   accounts, and CI credentials to the minimum access they need.
4. **Vet every new dependency** (package, GitHub Action, MCP server) per
   `docs/standards/dependency-management.md` before it is merged.
5. **Redact sensitive fields from logs by default**
   (`docs/standards/logging.md`).
6. **No security-sensitive logic duplicated ad hoc** — centralize
   authentication/authorization checks rather than reimplementing them per
   call site.
7. **Security-relevant fixes get a `memory/lessons-learned.md` entry**
   (`CLAUDE.md` §15) without exception.

## Design Decisions

- **This document holds the checklist; Volume 07 holds the reasoning and
  process** (threat model, disclosure process) — keeping the two separate
  lets this file stay short and directly usable in review.

## Best Practices

- Treat any code touching authentication, authorization, or cryptography
  as high-scrutiny in review, regardless of who (or what) wrote it.
- When adding a new external input source (endpoint, webhook, file upload),
  write down its trust boundary and validation rule before implementing
  the happy path.
- Review CI/automation credentials periodically for scope creep.

## Examples

See `docs/volumes/07-security.md` "Examples" section for a full webhook
signature-verification example.

## Common Mistakes

See `docs/volumes/07-security.md` "Common Mistakes" — this document does
not duplicate that list; consult it directly.

## Future Improvements

- Add automated secret-scanning and dependency vulnerability scanning to
  `.github/workflows/` once there is a codebase/manifest for them to scan.

## Related Documents

- `docs/volumes/07-security.md` — full reasoning, threat model, disclosure process
- `docs/standards/dependency-management.md`
- `docs/standards/logging.md`
- `CLAUDE.md` §7
