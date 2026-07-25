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

## Pattern: Verify a state-dependent UI against live data by borrowing a real state transition, not mocking one

**Used in**: browser-verifying a "reviewer queue" UI whose local seed data
was 100% in a terminal `published` state, so the empty-queue path was the
only thing naturally reachable.

**Shape**:
1. Identify one row of real (non-fake) data and use an admin/service-role
   credential to write it back to an earlier state in the same state
   machine the UI drives (e.g. `published` → `ai_generated`).
2. Exercise the actual UI against that row — not a mock, not a fixture
   file — so every rendering path, button, and API call under test is the
   real one.
3. Let the UI's own forward-moving actions (approve → publish, in this
   case) carry the row back to its original state, rather than hand-writing
   the original value back afterward. This is both cheaper (no separate
   restore step to get subtly wrong) and a stronger test — it exercises the
   state machine's actual transitions, not just static rendering.
4. Confirm the row landed back where it started before considering the
   verification complete.

**When to use**: verifying a review/moderation/approval-style UI (or any UI
gated on a specific record state) when the available seed/dev data doesn't
naturally include a record in the state you need to see — and the workflow
itself is capable of restoring the state you borrowed. Only do this against
local/dev data with a reversible, forward-path-restorable transition; never
against production data, and never if the workflow can't cleanly return the
record to its original state (in which case, do an explicit restore write
and verify it, rather than assuming the workflow will handle it).

## Pattern: Post-dispatch verification for subagents working in a git worktree

**Used in**: subagent-driven-development sessions where an implementer
subagent is dispatched into an isolated worktree (two occurrences in one
session — a subagent commit and a controller edit both landed in the wrong
checkout; see `memory/lessons-learned.md#LL-0022`).

**Shape**:
1. Every dispatch prompt's first instruction is a mandatory `pwd` + `git
   branch --show-current` check, with an explicit "stop and report if this
   doesn't match `<worktree-path>` / `<branch-name>`" — before the subagent
   touches any file.
2. Every file path handed to the subagent (or used by the controller
   itself) is the full worktree-absolute path, freshly derived from that
   session's own `pwd`, never typed from memory or reused from an earlier
   context.
3. After the subagent reports done, the controller independently checks
   `git log --oneline -1` in *both* the worktree and the main checkout —
   not just the worktree — before marking the task complete. A self-report
   of "committed" is not sufficient evidence of *where*.
4. If work landed in the wrong checkout, fix by cherry-pick (worktree) +
   hard-reset (main), not by re-doing the work.

**When to use**: any workflow that dispatches subagents (or resumes work
across `cd` boundaries) into a git worktree, container, or other
filesystem isolation that a shell `cd` enters but a tool's own path
resolver may not honor.

## Pattern: Diagnosing a stalled background subagent with evidence, not guesses

**Used in**: multiple subagent-driven-development tasks in one session
where a background subagent went quiet mid-task without a final report
(complements `memory/lessons-learned.md#LL-0014`).

**Shape**:
1. Before assuming a subagent is stuck, gather concrete evidence: `git
   status`/`git diff` in its worktree (is content actually changing?),
   `ps aux` for an active process matching its task (e.g. a running test
   runner), timestamps on recently modified files.
2. If the evidence shows real, ongoing progress, do nothing — let it
   continue; a background test run or long compile is not "frozen" just
   because no message has arrived yet.
3. If the evidence shows no progress for an extended period, resume the
   agent via a direct message to its own agent id (not a fresh dispatch,
   which loses its context) with a short nudge.
4. Report the concrete evidence found to whoever asked ("is it working?"),
   not a guess — e.g. "yes, `git diff` shows N new lines in the last
   minute" or "no process matching the test runner is running; nudging it."

**When to use**: any time a long-running background subagent (or any
async job whose completion notification might not reliably reach its own
next turn) appears idle and someone asks whether it's actually working.

## Status

_Last reviewed: 2026-07-25, after adding the worktree post-dispatch
verification and stalled-subagent diagnosis patterns above (from a
subagent-driven-development session building a new item-type feature).
Earlier note (repository bootstrap, 2026-07-01) about this file growing as
concrete code patterns emerge in `src/` still applies — no `src/` patterns
yet._
