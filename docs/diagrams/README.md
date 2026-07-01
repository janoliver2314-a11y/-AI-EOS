# Diagrams

Diagram sources and conventions for AI-EOS documentation.

## Convention

- **Format**: prefer [Mermaid](https://mermaid.js.org/) code blocks embedded
  directly in the relevant Markdown document, since GitHub renders Mermaid
  natively without extra tooling. Reserve this directory for diagrams that
  are reused across multiple documents or that require a format Mermaid
  can't express well.
- **Exported images** (PNG/SVG), if used, are stored here under a
  descriptive name matching the document that uses them
  (`<document-name>-<diagram-purpose>.svg`) and referenced by relative
  path from that document.
- Keep diagram source (Mermaid text or an editable source file) alongside
  any exported image so it can be updated, not just replaced.

## Current Diagrams

None yet. The four-layer architecture diagram in
`docs/volumes/02-architecture.md` is currently expressed as an inline
ASCII/text block; migrate it to Mermaid here if it grows complex enough to
need proper rendering.

## Related Documents

- `docs/volumes/02-architecture.md`
- `docs/architecture/overview.md`
