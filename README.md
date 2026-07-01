# AI-EOS — Artificial Intelligence Engineering Operating System

AI-EOS is a production-grade operating system for AI-assisted software
engineering: a documentation library, engineering standards, persistent
project memory, and automation extension points designed to keep a
codebase consistent, maintainable, and resilient to context loss — whether
that loss comes from a new AI session starting cold, a new engineer
joining, or six months passing between touches on a subsystem.

## Vision

Most AI-assisted development setups optimize for a single session's
output. AI-EOS optimizes for **consistency across an unbounded number of
future sessions** by treating documentation, standards, and memory as
load-bearing infrastructure rather than optional polish. See
`docs/volumes/01-foundation.md` and `docs/volumes/09-business-strategy.md`
for the full reasoning.

## Features

- **`CLAUDE.md`** — a comprehensive operating manual: mission, principles,
  coding/documentation/security standards, git workflow, planning
  methodology, and a Definition of Done. Authoritative for any session
  working in this repository.
- **A 10-volume documentation library** (`docs/volumes/`) covering
  foundation, architecture, memory, agents, workflow, engineering
  standards, security, deployment, and business strategy — each volume
  self-contained and cross-linked.
- **Checkable engineering standards** (`docs/standards/`) — one topic per
  file, covering naming, error handling, logging, testing, dependencies,
  versioning, performance, scalability, reliability, and security.
- **A persistent memory system** (`memory/`) that records architecture
  decisions, coding conventions, technology choices, lessons learned from
  bugs, the roadmap, and reusable patterns — so the same mistake is never
  repeated twice.
- **Reusable templates and prompts** (`templates/`, `prompts/`) for ADRs,
  bug reports, feature specs, postmortems, and AI-driven workflows.
- **Designed automation extension points** (`scripts/`, `.github/workflows/`)
  for CI/CD, documentation generation, and knowledge indexing — documented
  before they're built, so the first implementation has a spec to satisfy.

## Repository Layout

```
AI-EOS/
├── CLAUDE.md              # Operating manual — read this first
├── README.md              # This file
├── LICENSE                # MIT
├── CHANGELOG.md
├── CONTRIBUTING.md
├── CODE_OF_CONDUCT.md
│
├── docs/
│   ├── volumes/           # 10-volume documentation library
│   ├── architecture/      # Current-state architecture reference
│   ├── standards/         # Per-topic engineering standards
│   ├── playbooks/         # Step-by-step procedures
│   ├── diagrams/          # Diagram sources and conventions
│   ├── decisions/         # Architecture Decision Records
│   └── references/        # Glossary and external references
│
├── prompts/               # Reusable AI-assisted-work prompts
├── memory/                # Persistent project memory
├── templates/             # Reusable document templates
├── scripts/               # Automation extension points
├── src/                   # Application source (not yet populated)
├── tests/                 # Automated tests
├── assets/                # Static assets
│
└── .github/               # Issue/PR templates, CI extension points
```

## Quick Start

1. Read `CLAUDE.md` in full — it defines the rules everything else follows.
2. Read `docs/volumes/01-foundation.md` for the mission and philosophy.
3. Use `docs/volumes/10-master-reference.md` as your map to everything
   else — its "Quick Reference by Task" table gets you to the right
   document fast.
4. Before starting any task, check `memory/roadmap.md` for current
   priorities and `memory/lessons-learned.md` for relevant prior lessons.

## Development Workflow

1. Clarify the goal and survey `memory/` and `docs/decisions/` for existing
   context (`CLAUDE.md` §13).
2. Plan, implement, and test following `docs/standards/`.
3. Review against the checklist in `CLAUDE.md` §12.
4. Record any new decisions, lessons, or patterns in `memory/`.
5. Summarize the change, propose a commit message, and get explicit
   approval before committing or pushing (`CLAUDE.md` §10).

See `docs/volumes/05-workflow-engine.md` for the full workflow model and
where automation attaches to it.

## Documentation Map

| I want to... | Go to |
|---|---|
| Understand the operating rules | `CLAUDE.md` |
| Understand why AI-EOS exists | `docs/volumes/01-foundation.md` |
| Find any document quickly | `docs/volumes/10-master-reference.md` |
| Find a coding rule | `docs/standards/` |
| Record or find a decision | `docs/decisions/`, `memory/architecture-decisions.md` |
| Record or find a lesson from a bug | `memory/lessons-learned.md` |

## Contributing

See `CONTRIBUTING.md` for the full workflow, and `CODE_OF_CONDUCT.md` for
community standards. In short: read `CLAUDE.md`, follow the standards in
`docs/standards/`, add tests and documentation, and record lessons/decisions
in `memory/`.

## Roadmap

See `memory/roadmap.md` for the current, actively-maintained priority list,
and `docs/volumes/09-business-strategy.md` for the longer-term vision.

## License

MIT — see `LICENSE`.
