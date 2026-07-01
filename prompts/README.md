# Prompts

Reusable prompt templates for AI-assisted work in this repository. A prompt
here should encode enough context that it produces consistent, standards-
compliant output regardless of which session runs it.

## Available Prompts

| Prompt | Use for |
|---|---|
| `bug-fix.md` | Driving an AI session through the bug-fix playbook |
| `new-document.md` | Drafting a new document that complies with the documentation standard |

## Design Decisions

- Prompts reference `CLAUDE.md` and specific playbooks/standards rather
  than re-explaining process inline, so they stay accurate as those
  documents evolve.
- A prompt is added here once it has proven useful more than once, mirroring
  the bar for `memory/reusable-patterns.md`.

## Related Documents

- `docs/playbooks/`
- `CLAUDE.md`
