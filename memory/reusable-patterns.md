# Reusable Patterns

Patterns proven useful more than once, with a canonical example. A pattern
belongs here once it has been used a second time — a single use is not yet
a pattern (see `CLAUDE.md` §4, avoid premature abstraction).

## Pattern: Document Skeleton

**Used in**: every file under `docs/volumes/` and `docs/standards/`.

**Shape**: Purpose → Scope → Goals → Design Decisions → Best Practices →
Examples → Common Mistakes → Future Improvements → Related Documents.

**Canonical example**: `docs/standards/documentation.md` (defines the
pattern); `docs/volumes/01-foundation.md` (applies it to narrative content).

**When to use**: any new substantive document under `docs/`.

## Pattern: Indexed Summary + Detailed Source

**Used in**: `memory/architecture-decisions.md` (index) →
`docs/decisions/` (detail); `CLAUDE.md` (summary) → `docs/standards/`
(detail); `docs/volumes/10-master-reference.md` (index) → individual
volumes (detail).

**Shape**: a short, scannable index file that links to full detail, kept in
sync by treating the index update as part of the same change that adds the
detailed content.

**When to use**: whenever a growing collection of documents needs fast
lookup without forcing a reader through full detail every time.

## Pattern: Clone-Inspect-Decide before installing a third-party AI-agent tool

**Used in**: adding Headroom, UI UX Pro Max, and Stop Slop to a Claude Code
environment's shared "Frameworks" reference directory (three separate
occasions in one session).

**Shape**:
1. `git clone --depth 1` the repo into a scratch/temp directory — never
   assume the install method from the URL or repo name alone.
2. Read the README and manifest files (`package.json`,
   `.claude-plugin/plugin.json`, `SKILL.md`, `pyproject.toml`) to determine
   the *actual* supported install mechanism. These vary widely even across
   superficially similar "AI agent tool" repos: an npm-published CLI, a
   Claude Code plugin marketplace, a bare skill folder meant to be copied
   as-is, or a Python package fronted by a proxy/daemon.
3. Match install scope to actual risk. A global npm CLI or a static
   skill-folder copy can proceed directly. Anything that requires sudo,
   compiles native code, wraps another tool's launch behavior, or installs
   a persistent background service should be confirmed with the user
   first, with the tradeoffs stated plainly.
4. Verify the install actually works end-to-end (run a smoke command,
   check a live skill/tool list, hit a health endpoint) rather than
   trusting "install succeeded" output alone — see `LL-0003` in
   `memory/lessons-learned.md`.
5. Document the tool in a per-tool `README.md` under the shared reference
   directory (what it is, how it's installed/invoked, where its live
   config/state lives, any platform caveats hit during setup) and add one
   line to the top-level index pointing to it, in the same change.

**When to use**: any request to add/install a GitHub repo as a tool for an
AI coding agent environment.

## Status

_Last reviewed: 2026-07-03, after adding the Clone-Inspect-Decide pattern
above. Earlier note (repository bootstrap, 2026-07-01) about this file
growing as concrete code patterns emerge in `src/` still applies — no
`src/` patterns yet._
