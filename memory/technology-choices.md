# Technology Choices

What technologies AI-EOS uses and why, at a glance. Full vetting process:
`docs/standards/dependency-management.md`.

| Technology | Chosen for | Alternatives considered | Why |
|---|---|---|---|
| Markdown | All documentation | reStructuredText, AsciiDoc | Renders natively on GitHub, zero tooling required, universally readable |
| MIT License | Project licensing | Apache 2.0, proprietary | Maximally permissive for a methodology intended to be reused/adopted elsewhere |
| Semantic Versioning | Release versioning | CalVer, ad hoc | Standard, well-understood compatibility signaling once releases begin |

## Status

No application language, framework, database, or hosting platform has been
chosen yet. These decisions are deferred until a concrete application
requirement exists (`docs/volumes/02-architecture.md`), and will be
recorded here — with a corresponding ADR in `docs/decisions/` — once made.
