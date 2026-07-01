# Lessons Learned

Structured record of bugs and mistakes, per `CLAUDE.md` §15. Every entry
follows the same shape: Root Cause, Why It Happened, Solution, Preventive
Rule, Similar Situations. Search this file before writing new code in an
unfamiliar area — never knowingly repeat a documented mistake.

Entries are numbered `LL-NNNN`, sequential, never renumbered or deleted.

---

### LL-0001 — Untrusted webhook payloads must be authenticated before parsing

- **Root Cause**: A hypothetical/example webhook handler pattern parsed and
  acted on payload fields before verifying the request's signature,
  meaning an attacker-controlled payload could influence behavior before
  authenticity was established.
- **Why It Happened**: Signature verification is easy to treat as "one more
  step" and insert after the main logic is written, rather than designing
  it as the first gate.
- **Solution**: Established the rule that signature/authenticity
  verification must be the first operation in any external-input handler,
  before deserialization or field access.
- **Preventive Rule**: Added to `docs/standards/security.md` and
  `docs/volumes/07-security.md`: verify authenticity before parsing content
  for any externally-triggered handler (webhooks, callbacks, uploads).
- **Similar Situations**: Any future webhook receiver, OAuth callback, or
  file upload handler — check `docs/standards/security.md` before merging.

<!--
Template for new entries — copy this block:

### LL-NNNN — Short, specific title

- **Root Cause**:
- **Why It Happened**:
- **Solution**:
- **Preventive Rule**:
- **Similar Situations**:
-->
