# Volume 07 — Security

## Purpose

Define AI-EOS's security posture: the threat model it designs against, its
core security principles, and the process for handling vulnerabilities.

## Scope

Covers security principles applicable to the repository itself (secrets,
dependencies, contribution process) and the baseline expectations for any
application code built under `src/`. Detailed, checkable rules live in
`docs/standards/security.md`; this volume explains the reasoning and
process.

## Goals

- Make secure-by-default the path of least resistance, not an extra step.
- Ensure vulnerabilities have a clear, safe disclosure and response path
  from day one, even before a public application exists.
- Keep the threat model explicit so security reviews have something
  concrete to check against.

## Threat Model (Baseline)

Even before application code exists, the repository itself has a threat
surface:

- **Secrets exposure**: credentials or tokens committed to git history,
  logs, or documentation examples.
- **Supply chain risk**: dependencies (npm packages, GitHub Actions,
  MCP servers) with malicious or compromised code.
- **Untrusted external input**: once `src/` exists, any user input, webhook
  payload, or third-party API response must be treated as untrusted until
  validated.
- **Overprivileged automation**: CI/CD or bot credentials with more access
  than the automation actually needs.

## Design Decisions

- **Least privilege by default** for every credential, token, and service
  account — request the minimum scope needed, and document why if broader
  access is ever required.
- **No secrets in git, ever** — not in commits, not in commit history, not
  in example configuration files. Use `.env.example` with placeholder
  values and document required variables.
- **New dependencies are vetted before merge** — see
  `docs/standards/dependency-management.md`. This applies equally to npm
  packages, GitHub Actions, and MCP servers, all of which execute code with
  some level of trust.
- **Security-relevant fixes always produce a `memory/lessons-learned.md`
  entry**, per `CLAUDE.md` §15 — security mistakes are exactly the class of
  bug most likely to recur silently if not documented.

## Best Practices

- Validate and sanitize all external input at the boundary where it enters
  the system; never assume a caller (internal or external) has already
  validated it unless that contract is explicit and tested.
- Redact secrets and sensitive fields from logs by default (see
  `docs/standards/logging.md`).
- Review any new automation (CI workflow, script, MCP server) for what
  credentials it needs and whether that scope is minimal.
- Treat AI-generated code touching authentication, authorization, or
  cryptography with extra scrutiny in review — these are high-consequence,
  low-forgiveness areas.

## Examples

Adding a webhook receiver:

1. Verify the payload signature **before** parsing or acting on its
   contents (see `memory/lessons-learned.md` LL-0001 for why this ordering
   matters).
2. Validate the payload shape against an explicit schema; reject anything
   that doesn't match rather than defensively coercing it.
3. Log the event with sensitive fields redacted.
4. Document the required signing secret in `.env.example`, never commit its
   value.

## Common Mistakes

- Committing a `.env` file "temporarily" — git history is permanent even
  after the file is later removed; treat any committed secret as
  compromised and rotate it.
- Adding a dependency or GitHub Action from an unfamiliar publisher without
  checking its maintenance status or permissions footprint.
- Logging full request/response bodies that may contain credentials or PII.
- Writing a fallback that silently swallows an authentication failure
  instead of rejecting the request loudly.

## Responsible Disclosure

Security issues should not be filed as public GitHub issues. Until a
dedicated security contact/process is established, report suspected
vulnerabilities by opening a private security advisory on the repository
(GitHub's "Report a vulnerability" flow) or contacting a maintainer
directly. Do not include exploit details in a public channel.

## Future Improvements

- Add automated dependency vulnerability scanning to `.github/workflows/`
  once a package manifest exists (see `docs/volumes/05-workflow-engine.md`
  extension points).
- Formalize a `SECURITY.md` with a specific contact and response-time
  commitment once the project has a stable maintainer group.

## Related Documents

- `CLAUDE.md` §7 — security principles summary
- `docs/standards/security.md` — detailed, checkable security rules
- `docs/standards/dependency-management.md` — dependency vetting process
- `memory/lessons-learned.md` — recorded security-relevant incidents
