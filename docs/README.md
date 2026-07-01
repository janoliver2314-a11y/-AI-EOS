# AI-EOS Documentation

This is the entry point to AI-EOS's documentation system. Start with
`CLAUDE.md` at the repository root for the operating rules, then use this
map to find what you need.

## Structure

| Directory | Contents |
|---|---|
| `volumes/` | The 10-volume narrative documentation set — start here for "why" |
| `architecture/` | Concrete, current-state architecture references |
| `standards/` | Per-topic, checkable engineering standards — the "how" |
| `playbooks/` | Step-by-step procedures for recurring operational tasks |
| `diagrams/` | Diagram sources and conventions |
| `decisions/` | Architecture Decision Records (ADRs) |
| `references/` | External reference material and glossaries |

## Where to Start

- **New to this repository?** Read `CLAUDE.md`, then
  `docs/volumes/01-foundation.md`, then `docs/volumes/10-master-reference.md`.
- **Looking for a specific rule?** Check `docs/volumes/10-master-reference.md`'s
  "Quick Reference by Task" table, or search `docs/standards/` directly.
- **Making an architectural change?** Read `docs/volumes/02-architecture.md`
  and check `docs/decisions/` for prior related decisions.
- **Fixing a bug?** Check `memory/lessons-learned.md` first, then follow
  `CLAUDE.md` §15 to record what you find.

## Documentation Standards

Every substantive document here follows the skeleton defined in
`docs/standards/documentation.md`: Purpose, Scope, Goals, Design Decisions,
Best Practices, Examples, Common Mistakes, Future Improvements, Related
Documents. This keeps documents comparable and complete.
