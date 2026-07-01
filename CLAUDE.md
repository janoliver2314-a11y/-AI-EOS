# CLAUDE.md — AI-EOS Operating Manual

> This file is the permanent operating manual for any Claude Code session (or human contributor) working in this repository. It is authoritative: when other documents conflict with this file, this file wins unless a document explicitly states it supersedes CLAUDE.md for its domain. Read this file in full before making non-trivial changes.

## 1. Project Mission

AI-EOS (Artificial Intelligence Engineering Operating System) is a production-grade operating system for AI-assisted software engineering. It is not a single tool — it is the combination of:

- **Documentation** that captures how and why the system works the way it does.
- **Standards** that make output from any contributor (human or AI) consistent and predictable.
- **Memory** that accumulates decisions and lessons so mistakes are not repeated.
- **Workflow structures** (agents, playbooks, automation extension points) that let engineering work scale beyond a single person or a single session.

The long-term goal is a repository that can onboard a new engineer — or a new AI session with zero prior context — and have them productive and aligned with existing conventions within minutes, not days.

## 2. Development Philosophy

1. **Design for the next session, not just this one.** Every session (human or AI) starts cold. The repository's documentation and memory are the only continuity that exists. Write as if the next reader has no access to this conversation.
2. **Optimize for maintainability over cleverness.** Boring, obvious code that a tired engineer can understand at 2am beats a clever one-liner.
3. **Prefer explicit over implicit.** Hidden assumptions are the single largest source of long-term bugs and architectural drift. State assumptions in writing.
4. **Modular over monolithic.** Systems, documents, and code should be composed of small, independently understandable units with clear boundaries.
5. **Every mistake is a system improvement opportunity.** A bug fixed without a recorded lesson is a bug that will happen again, possibly to someone else.
6. **Do the smallest thing that is actually correct.** Do not build speculative abstractions for requirements that do not yet exist (see YAGNI in §4).

## 3. Repository Conventions

### 3.1 Directory Structure

```
AI-EOS/
├── CLAUDE.md              # This file — operating manual
├── README.md              # Public-facing project overview
├── LICENSE                # MIT license
├── CHANGELOG.md           # Keep a Changelog format
├── CONTRIBUTING.md        # Contribution workflow
├── CODE_OF_CONDUCT.md     # Community standards
│
├── docs/                  # All long-form documentation
│   ├── volumes/           # The 10-volume master documentation set
│   ├── architecture/      # System architecture references
│   ├── standards/         # Engineering standards (one topic per file)
│   ├── playbooks/         # Step-by-step operational procedures
│   ├── diagrams/          # Diagram sources and conventions
│   ├── decisions/         # Architecture Decision Records (ADRs)
│   └── references/        # External reference material and glossaries
│
├── prompts/               # Reusable prompt templates for AI-assisted work
├── memory/                # Persistent project memory (decisions, lessons, roadmap)
├── templates/             # Reusable document/code templates
├── scripts/               # Automation scripts and extension points
├── src/                   # Application source code
├── tests/                 # Automated test suites
├── assets/                # Static assets (images, diagrams, exports)
│
└── .github/               # GitHub-native automation and templates
    ├── workflows/         # CI/CD pipeline definitions
    ├── ISSUE_TEMPLATE/    # Issue templates
    └── PULL_REQUEST_TEMPLATE.md
```

### 3.2 File Naming

- Markdown documents: `kebab-case.md` (e.g. `error-handling.md`).
- Volumes: `NN-topic-name.md` with a two-digit zero-padded prefix for stable ordering.
- ADRs: `NNNN-short-title.md` with a four-digit zero-padded sequence number, never reused.
- Source code naming is defined per-language in `docs/standards/naming-conventions.md`.

### 3.3 Where New Content Goes

| Content type | Location |
|---|---|
| A decision that affects architecture | `docs/decisions/` (ADR) + summary in `memory/architecture-decisions.md` |
| A reusable coding pattern | `memory/reusable-patterns.md` |
| A bug fix with a root cause | `memory/lessons-learned.md` |
| A new engineering rule | `docs/standards/` |
| A step-by-step operational procedure | `docs/playbooks/` |
| A prompt used to drive AI-assisted work | `prompts/` |
| Conceptual/narrative documentation | `docs/volumes/` |

## 4. Engineering Principles

- **YAGNI** — You Aren't Gonna Need It. Do not build for hypothetical future requirements.
- **DRY, but not at the cost of clarity** — three similar lines are better than a premature abstraction; a fourth repetition is a signal to reconsider.
- **Single Responsibility** — a module, function, or document should have one reason to change.
- **Fail loudly, fail early** — errors should surface as close to their cause as possible, with enough context to diagnose without reproducing.
- **Boundaries are sacred** — validate and sanitize at system boundaries (user input, external APIs, file I/O); trust internal, already-validated data.
- **No silent fallbacks** — a fallback that hides a real failure is a bug waiting to be discovered in production.

## 5. Coding Standards

Full detail lives in `docs/standards/`. Summary rules that apply regardless of language:

1. Code must be self-documenting through naming; comments explain **why**, never **what**.
2. No dead code, no commented-out code blocks, no TODO comments without a linked issue.
3. No premature optimization; optimize only with a measured bottleneck (see `docs/standards/performance.md`).
4. Every public function/module has a defined error contract (what it throws/returns on failure).
5. Secrets, credentials, and tokens are never committed. Use environment variables and document required variables in `.env.example`.
6. Dependencies are added deliberately — see `docs/standards/dependency-management.md` before adding any new package.

## 6. Documentation Standards

Every substantive document in this repository (volumes, standards, ADRs, playbooks) follows the same section skeleton, defined in full in `docs/standards/documentation.md`:

`Purpose → Scope → Goals → Design Decisions → Best Practices → Examples → Common Mistakes → Future Improvements → Related Documents`

Rules:

- Write for a reader with zero prior context.
- Link related documents explicitly rather than assuming discoverability.
- Update a document in place rather than creating a competing "v2" file. Use `CHANGELOG.md` and git history for evolution tracking.
- Never leave a placeholder section ("TBD", "Coming soon") in a merged document. If a topic isn't ready, don't create the document yet.

## 7. Security Principles

1. Treat all external input (user input, API responses, file uploads, webhook payloads) as untrusted until validated.
2. No secrets in source control, commit history, logs, or error messages.
3. Principle of least privilege for all credentials, tokens, and service accounts.
4. Dependencies are checked for known vulnerabilities before being added and periodically thereafter.
5. Authentication and authorization logic is never duplicated ad hoc — centralize it.
6. Full detail and threat-model methodology: `docs/volumes/07-security.md` and `docs/standards/security.md`.

## 8. Performance Expectations

- Correctness first, then clarity, then performance. Do not trade away the first two without a measured need.
- Any performance-motivated change must cite a benchmark or profiling result in its PR description or commit message.
- Performance-sensitive paths must have a regression test or benchmark checked into `tests/`.
- Full detail: `docs/standards/performance.md`.

## 9. Testing Requirements

- New behavior ships with tests that would fail without the change.
- Bug fixes ship with a regression test that reproduces the original bug.
- Test names describe the behavior under test, not the implementation.
- No test is skipped or marked pending without a linked tracking issue and a reason in a comment.
- Full detail: `docs/standards/testing.md`.

## 10. Git Workflow

1. All work happens on a feature branch; the default branch is protected and reflects production-ready state.
2. Commits are small, logically scoped, and buildable/testable in isolation where practical.
3. **Never push automatically without explicit approval for this project's bootstrap and structural work.** For a Claude Code session: after completing a logical unit of work, summarize the changes, propose a commit message, and wait for explicit user approval before committing or pushing — unless the user has separately authorized autonomous commits/pushes for a specific task.
4. Never force-push over another contributor's work without confirmation.
5. Never use `--no-verify` to bypass hooks; fix the underlying failure instead.

## 11. Commit Standards

- Format: a short imperative summary line (≤ 72 chars), optionally followed by a blank line and a body explaining **why**.
- Reference the ADR, issue, or lesson entry that motivated the change where applicable.
- One logical change per commit. Do not mix refactors with behavior changes.

Example:

```
Add retry policy for MCP tool calls

Transient network errors were surfacing as hard failures with no
recovery path (see memory/lessons-learned.md#LL-0003). Adds bounded
exponential backoff per docs/standards/error-handling.md.
```

## 12. Review Checklist

Before considering any change complete, verify:

- [ ] Does it match the relevant standards in `docs/standards/`?
- [ ] Is there a test that would fail without this change?
- [ ] Are new public interfaces documented?
- [ ] Does it introduce a new dependency? If so, is it justified per `docs/standards/dependency-management.md`?
- [ ] Does it touch security-sensitive code? If so, has the threat model been reconsidered?
- [ ] Is there duplication that should be consolidated?
- [ ] Does this change contradict an existing ADR? If so, has a new ADR been written to supersede it?
- [ ] Have relevant `memory/` files been updated (lessons learned, roadmap, patterns)?

## 13. Planning Methodology

For any non-trivial task:

1. **Clarify the goal** — restate the problem in your own words; identify what "done" looks like.
2. **Survey existing context** — check `memory/`, `docs/decisions/`, and relevant `docs/volumes/` before proposing a new approach.
3. **Identify the smallest correct solution** — resist scope expansion.
4. **Consider second-order effects** — what does this change make harder or easier later?
5. **Write the plan down** for anything touching more than a few files or introducing a new pattern.
6. **Execute incrementally**, validating at each step rather than batching all changes before the first check.

## 14. Decision-Making Framework

Use an ADR (`docs/decisions/`) whenever a decision:

- Is expensive or awkward to reverse.
- Affects more than one module/domain.
- Chooses between two or more viable technical approaches.
- Establishes a convention future work is expected to follow.

An ADR records: **Context**, **Decision**, **Alternatives Considered**, **Consequences**. ADRs are never deleted; a changed decision gets a new ADR that supersedes the old one (linked both directions).

## 15. Learning System

Before implementing a feature or fixing a bug, consult the relevant files in `memory/` for prior decisions, lessons learned, and known pitfalls. When new knowledge is gained, update the appropriate memory document as part of the same change.

Whenever a bug is fixed, before closing the task, append an entry to `memory/lessons-learned.md` recording:

- **Root Cause** — the actual underlying cause, not the symptom.
- **Why It Happened** — what process or knowledge gap allowed it.
- **Solution** — what was changed.
- **Preventive Rule** — a concrete, checkable rule that would have caught this earlier (add it to a standard or checklist if generally applicable).
- **Similar Situations** — where else this class of bug could exist.

Before writing new code in an area, search `memory/lessons-learned.md` and `memory/reusable-patterns.md` for relevant prior entries. **Never knowingly repeat a mistake that has an existing lesson entry.**

## 16. Error Prevention Rules

- Never guess at an API, config key, or file path — verify it exists in the codebase or its documentation first.
- Never delete or overwrite files without confirming they aren't in-progress work (check git status/history).
- Never introduce a breaking change to a public interface without a corresponding ADR and CHANGELOG entry.
- Never mark a task complete with failing tests, partial implementation, or unresolved TODOs.
- When uncertain between two approaches with materially different tradeoffs, ask rather than silently picking one.

## 17. Definition of Done

A unit of work is **Done** only when all of the following are true:

- [ ] The stated requirement is fully implemented — no partial implementation, no placeholder content.
- [ ] Tests exist and pass for the new/changed behavior.
- [ ] Documentation affected by the change has been updated (not just added — updated in place where relevant).
- [ ] The Review Checklist (§12) has been satisfied.
- [ ] Any new lesson, decision, or pattern has been recorded in `memory/`.
- [ ] The change has been summarized in human-readable terms for the user, with a proposed commit message, and explicit approval has been obtained before committing/pushing (per §10).

## 18. Related Documents

- `docs/volumes/01-foundation.md` — expanded mission and philosophy
- `docs/volumes/06-engineering-standards.md` — expanded engineering standards
- `docs/standards/` — per-topic standards referenced throughout this file
- `memory/` — persistent project memory
- `docs/decisions/` — architecture decision records
