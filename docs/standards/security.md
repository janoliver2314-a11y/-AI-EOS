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
8. **Never pass a secret as a command-line argument.** A process's `argv` is
   readable by anything running as the same user (`ps`, and `/proc/*/cmdline`
   on Linux) for as long as the process lives, so `-H "Authorization: Bearer
   $TOKEN"` publishes the token to the machine for the duration of the request.
   Prefer, in order: the tool's own stdin/config-file mechanism (`curl --config
   -`, `gh` reading `GH_TOKEN`, `ssh-add`), then an environment variable scoped
   to the single call, then a file with restrictive permissions. Rule 1 keeps
   secrets out of the repo and rule 5 keeps them out of logs; this keeps them
   out of the third channel, which is the one people forget (see
   `memory/lessons-learned.md#LL-0093`).

9. **A command that writes a secret to a file must use an absolute path.** A
   relative `>> .env` resolves against whatever directory the shell happens to
   be in, so the same command that configures a project can instead deposit a
   live production credential in `$HOME` — outside the repo, outside its
   `.gitignore`, and outside the place anyone will think to look for it. Give
   the full path, and verify where the value landed rather than that the command
   exited zero.

10. **Validate a credential's shape, not just its presence, wherever one is read
    from configuration.** Most providers use a recognizable prefix, so the check
    costs one comparison and catches the whole class of plausible non-secrets:
    masked values, `[SENSITIVE]`-style placeholders from secret managers that
    refuse to return a value, unresolved references, and `.env.example`
    defaults. A check for non-empty tests only that someone set something (see
    `memory/lessons-learned.md#LL-0107`).

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
