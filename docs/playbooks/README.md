# Playbooks

Step-by-step operational procedures for recurring tasks. Where
`docs/volumes/` explains *why* the system works a certain way, playbooks
tell you exactly *what to do*, in order, for a specific recurring
situation.

## Available Playbooks

| Playbook | Use when |
|---|---|
| `bug-fix.md` | Fixing a reported or discovered bug |
| `new-standard.md` | Proposing a new or changed engineering standard |
| `onboarding-new-session.md` | Starting a new AI session or onboarding a new contributor with zero context |

## Design Decisions

- Playbooks are deliberately narrow and procedural — if a playbook needs to
  explain *why* a step exists in depth, that reasoning belongs in the
  relevant `docs/volumes/` document, linked from the playbook.
- A playbook is added once a procedure has been performed at least twice in
  a similar way; a one-off procedure doesn't need a playbook.

## Related Documents

- `docs/volumes/05-workflow-engine.md`
- `CLAUDE.md` §13, §15, §17
