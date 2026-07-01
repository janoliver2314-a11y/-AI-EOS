# Project Memory

This directory is AI-EOS's persistent, working memory — distinct from
`docs/`, which describes how the system is intended to work. Memory records
what has actually happened: decisions made, conventions in force, lessons
learned, and where the project is headed.

See `docs/volumes/03-memory-system.md` for the full design rationale.

## Files

| File | Contents | Update when |
|---|---|---|
| `architecture-decisions.md` | Index of ADRs with one-line summaries | An ADR is created or superseded |
| `coding-conventions.md` | Current, concrete coding conventions in force | A convention is established or changed |
| `technology-choices.md` | What technologies were chosen and why | A dependency or technology is adopted |
| `lessons-learned.md` | Root-cause entries for bugs and mistakes | A bug is fixed (`CLAUDE.md` §15) |
| `roadmap.md` | Near/mid-term plan, in priority order | Priorities shift |
| `future-ideas.md` | Deferred, unprioritized ideas | An idea is considered but not scheduled |
| `reusable-patterns.md` | Patterns proven useful more than once | A pattern is reused for the second time |

## Retrieval Discipline

Before starting non-trivial work, check the files above relevant to your
task — especially `lessons-learned.md` and `reusable-patterns.md` — per
`CLAUDE.md` §15. Never knowingly repeat a documented mistake.
