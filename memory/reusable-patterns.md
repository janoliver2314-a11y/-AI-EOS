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

## Pattern: Prove a new test discriminates by breaking the implementation, not by deleting it

**Used in**: two fix waves in one subagent-driven feature branch, each
adding tests to close a gap a reviewer had found. Operationalizes
`memory/lessons-learned.md#LL-0028` (where "fails before the fix" evidence
turned out to come from a missing import rather than the defective
behavior) and rule 7 in `docs/standards/testing.md`.

**Shape**:
1. Write the new test against the fixed code and watch it pass. This
   proves nothing yet — it is the starting point, not the evidence.
2. **Reintroduce the defect in the implementation**, as narrowly as
   possible: change the one expression, swap the two arguments, restore
   the old copy string. Do not stash, delete, or rename the file — a
   test that fails on a missing import has told you only that the import
   is missing.
3. Run the test and record *which* assertions fail and with what message.
   Exactly the intended ones should fail. If none fail, the test does not
   discriminate and the gap is still open. If more fail than expected,
   you have learned the blast radius of that expression.
4. Revert the deliberate break and confirm a clean `git diff` on the
   implementation file before committing — the revert is the step most
   likely to be forgotten, and it ships the defect.
5. Report the observation, not the intention: "with the count computed
   globally, 1 of 10 failed — the second-tab assertion" beats "verified
   the test catches it."

**Why it works**: the failure mode this guards against is a test that
passes for both the right and the wrong implementation, which is
invisible from a green run and from a red run caused by anything other
than behavior. Only varying the behavior while holding everything else
fixed separates the two.

**Payoff observed**: in one case a whole pre-existing suite proved
entirely blind to an argument swap — reversing `(given, correct)` in a
scoring dispatcher failed exactly 1 of 66 tests, the newly added one.
Without step 2 that suite would have been assumed to cover the branch's
central invariant.

**When to use**: any test written to close a reviewer-found gap, any
regression test for a bug being fixed, and especially any test whose
fixtures are symmetric or whose assertions are negative
(`not.toHaveTextContent`, `assert x not in …`) — negative assertions go
vacuous silently when the thing they name is reworded or removed.

## Pattern: Credential handoff when Claude Code can't paste secrets itself

**Used in**: a Clerk + Google OAuth production cutover, where a secret key, an
OAuth client ID/secret pair, and even a `{"role": "reviewer"}` metadata JSON
blob all needed entering into third-party dashboards.

**Shape**: Claude Code is policy-blocked from typing or pasting any
credential-shaped value into a field — not just obvious secrets (API keys,
passwords) but anything a permission classifier reads as credential- or
permission-adjacent, including a bare JSON object like
`{"role": "reviewer"}`. The reliable handoff:
1. Claude finds/reveals the value at its source (e.g. clicks a dashboard's own
   "copy" button) and copies it to the clipboard — this is a *read*, not an
   *entry*, and is fine.
2. Claude tells the user exactly which field is open and focused, and that
   the value is on the clipboard.
3. The user pastes and saves. Claude does not touch the keyboard for that
   field, ever, including on its own attempt to "just try" — that attempt
   will be denied by the classifier anyway.
4. Claude verifies success via a side-channel signal (the row's "last
   updated" timestamp, a live end-to-end check downstream) — never by
   re-reading the secret itself.

**The gotcha this pattern guards against**: if Claude performs *any other
clipboard-copy action* between step 1 and the user's paste in step 3 —
including an incidental click that happens to copy an unrelated field's
value — the clipboard is silently overwritten and the user pastes the wrong
thing with no error surfaced anywhere. This happened once: an incidental
click on an old field's row copied its stale value over a freshly-copied
secret, and the mistake was only caught afterward by comparing "last
updated" timestamps across the two related fields. **Once a secret is on the
clipboard for a user to paste, do not perform any further clipboard-writing
action until the paste is confirmed.**

**When to use**: any task that involves entering an API key, secret, OAuth
credential, or any field a permission classifier might treat as
credential-adjacent (this can include role/permission JSON, not just literal
tokens) into a third-party dashboard on the user's behalf.

## Pattern: When a full verification would relay an infeasible amount of data through tool calls, ask the user instead of forcing it

**Used in**: verifying that 200 rows of cloud-published content matched a
local seed file byte-for-byte, to detect any reviewer edits made during a
manual review pass.

**Shape**: The established method for this class of check (proven at 30-row
scale in an earlier session) was to compute a content hash locally and
cloud-side and compare them. At 200 rows the local content was >1MB of SQL
text — relaying that through a tool-call parameter to a remote SQL executor
(no direct DB connection available) would have meant generating that entire
payload as model output: impractical and wasteful. Rather than attempting a
degraded version of the check (hash only a few fields and hope nothing else
changed) or silently skipping verification, the resolution was to ask the
user directly: "did you edit any content during this review pass?" One
direct question replaced an infeasible mechanical verification, and was
strictly more reliable than a partial hash check would have been.

**When to use**: any verification task where the thorough/mechanical
approach scales past what a single tool call or the current context budget
can carry, *and* the fact being verified is something the user directly
knows (did you edit X, did you change Y). Prefer asking over silently
degrading or skipping the check. Not a substitute for verification when the
user might not know the answer — "is the cloud data internally consistent"
is not something to just ask about — it applies specifically when the user
is the authoritative source for the fact in question.

## Pattern: Freeze production-proven code during adjacent work, then unfreeze it deliberately with its own re-proof

**Used in**: an eight-task branch adding a second managed service to an ops
toolkit that already upgraded a production service backing a live campaign.
The existing upgrade script had passed a two-scenario production rollback
drill; the new work needed to sit beside it without endangering it.

**Shape**:
1. **Name the frozen artifact explicitly in the plan's global constraints**,
   not in prose a task author might skim: "`upgrade-n8n.sh` is frozen for
   Tasks 1-7." Repeat it in every task dispatch that touches the directory.
2. **State the freeze in the exit criteria** so it is checked mechanically —
   `git diff <base> -- upgrade-n8n.sh` must be empty. Reviewers verified this
   every task; it caught nothing, which is the point.
3. **Let the frozen file's defects accumulate as recorded findings** rather
   than fixing them in passing. When the same defect was found and fixed in
   the *new* script (a safety-net flag armed one line too late), it was logged
   against the frozen one instead of being fixed there.
4. **Unfreeze in its own task**, whose deliverable is the change *and* the
   re-proof — not as a step inside a task about something else.
5. **Sequence the re-proof so the new consumer validates the shared code
   first.** The extracted helpers were proven by the new script's drill before
   the frozen script was moved onto them, so a failure would implicate the
   extraction rather than the production path.
6. **Gate the destructive re-proof on the operator.** Re-running a drill
   against live production is their call, not the agent's; the task stops and
   reports rather than deciding.

**Why**: the alternative — "just fix it while we're in here" — spends the
credibility of a drill that was expensive to run. Freezing converts every
temptation into a logged finding, and the unfreeze task then has a single,
reviewable diff whose blast radius is obvious.

**Trade-off**: the frozen file carries known defects for the duration, and
duplication accumulates in the meantime. Accept it only when the frozen
artifact's proof is genuinely costly to regenerate (a production drill, a
certification, a long soak). For ordinary code, fix it as you go.

---

## Pattern: Extract the shared abstraction after the second consumer exists, not before

**Used in**: the same branch. The plan originally accepted duplication
between two upgrade scripts to avoid touching the proven one; the operator
chose extraction instead. The sequencing question — extract first, or write
the second consumer and then extract — turned out to matter more than the
decision itself.

**Shape**:
1. Write the second consumer **standalone**, mirroring the first where the
   behavior genuinely matches. Do not reach for the abstraction yet.
2. Harden it independently. This is where the two diverge honestly: the new
   script grew bounded subprocess calls, process-group kills, and a distinct
   pull-failure path that the original had no need for.
3. **Extract only blocks that are identical modulo names.** A helper with a
   boolean flag that switches between two behaviors is worse than two clear
   call sites — say so in the task brief, because an eager implementer will
   otherwise unify the divergences too.
4. Move the *new* consumer over first and re-run its full proof. Commit that
   separately, so the older consumer's move can be reverted on its own.
5. Move the older consumer, in its own commit, then re-prove it.
6. **Record the surviving divergences and why**, in the shared library's
   header. The next reader's default assumption is that two scripts sharing a
   lib behave alike; only a written note corrects it.

**Why**: extracting from one consumer means guessing which parts are general.
Here the guess would have been wrong twice — the brief asserted the helpers
already lived in the shared lib when they were in fact private to one script
with a caller-specific path hardcoded, and the "identical" blocks turned out
to differ in four ways discovered only by writing the second consumer first.

**Trade-off**: you write some code twice and refactor it once. That cost is
real but bounded, and it is far smaller than an abstraction shaped around a
single example that then has to be widened under a deadline.

---

## Status

_Last reviewed: 2026-08-14, after adding the freeze-and-unfreeze and
extract-after-the-second-consumer patterns (from a subagent-driven branch that
added a self-hosted vector store beside a production-proven upgrade script).
Previously reviewed 2026-08-03, after adding the credential-handoff and
ask-before-relaying-infeasible-data patterns (from a domain/Clerk production
cutover session that also closed out a 200-item bank review). Previously
reviewed 2026-07-27, when the discriminating-test pattern was added (from a
subagent-driven-development session building the last of five item types).
Earlier: 2026-07-25 (worktree post-dispatch verification, stalled-subagent
diagnosis) and 2026-07-01 (repository bootstrap) — the bootstrap note about
this file growing as concrete code patterns emerge in `src/` still applies,
no `src/` patterns yet._
