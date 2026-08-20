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

**When a mutant survives, read the arrangement before the assertion.** Step
3 says a surviving mutant means the test does not discriminate; it does not
say where the blindness lives, and the instinct is to strengthen the
assertion. Often the fault is in the setup, which decided what was
observable before any assertion ran — most reliably when the test resets
the very state the mutant corrupts. A pair like arm/disarm tested as
*arm(x) → disarm → arm()* cannot see a mutant that makes `arm()` inherit
the previous value, because `disarm` already restored the default: the two
implementations produce identical state. Invoke the setter twice with
nothing in between (`docs/standards/testing.md` rule 11,
`memory/lessons-learned.md#LL-0095`). Framework-level resets — `beforeEach`,
fresh fixtures, auto-reset mocks — apply the same masking to every test in
the file without appearing in any of them.

**Mutate the defensive lines you added on your own initiative, not only the
behaviour you were asked to implement.** Steps 1-5 assume the test exists to
close a known gap, so a surviving mutant reads as a weak test. There is a
second case, and it is easier to miss: code you wrote proactively — a guard, a
bounds check, a fallback, a sanitising lookup — that no test ever asked for.
Its whole purpose is a case that does not arise in normal use, which is exactly
why the feature tests cannot reach it. Mutating it does not reveal a weak test;
it reveals **no test at all**, for a line already written and about to be
committed as though justified.

Observed: a filter dispatcher was refactored from a `switch` to a lookup table,
and the lookup deliberately used `hasOwnProperty` rather than
`TABLE[key]`, because the key is a free string and a plain lookup finds
`Object.prototype.toString` — truthy, and returning a truthy string, so every
record would match. Reverting that one call to a plain lookup broke nothing:
three other mutants had each killed their intended test, and this one killed
none. The reasoning behind the guard was sound and could be explained on
demand; that is precisely the trap, because **being able to explain a line is
not evidence that anything checks it**. The test written afterwards fails when
the guard is removed, which is what makes the guard's presence defensible.

The cheap discipline: after a change, list the lines you added that no
requirement named, and mutate each one. If a mutant survives, you have either
dead code or an untested guard — both worth knowing before the commit, and
neither visible in a green run.

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
   otherwise unify the divergences too. "Identical modulo names" includes
   **configuration** names: a block that reads one consumer's global by name is
   identical to its sibling in every way that matters, and the reference is
   removed by making it a parameter, not by declaring the block unshareable.
   Getting this backwards is how a safety fix lands in one of the two places
   that need it (`memory/lessons-learned.md#LL-0091`).
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

**The failure mode on the other side of this trade-off**, learned later on the
same pair of scripts: waiting for the second consumer is right, but the wait
has to *end*. Once both consumers exist and one of them gets a safety-critical
fix, "we deliberately kept these separate" turns from a sound decision into the
reason the sibling keeps the bug. When hardening a shared-shape path, the
trigger to revisit extraction is the fix itself — grep for siblings with the
same exposure before choosing where it lives (`LL-0091`).

---

## Pattern: Extend a field's encoding, not its type, when a consumer you cannot cheaply redeploy reads it directly

**Used in**: Task Command's `delegatedTo` — kept as a comma-separated string so
a live n8n Code node reading `` `with ${t.delegatedTo}` `` kept working, then
extended twice inside that encoding (multiple people, then CSV quoting for
names containing the separator); the same reasoning already governed meeting
`attendees` and minutes `recipients`. Also `recurring`, where a recurrence
*grammar* (`w:2:3`, `m:1:30`) was put inside the existing string field rather
than replacing it with an object.

**Shape**: before changing a field's type, enumerate every reader. If any of
them lives outside the deploy unit — an automation-platform node, a saved
query, a spreadsheet formula, a partner integration — treat the *type* as
frozen and add capability inside the *encoding* instead. Keep parse and format
in one library with its own tests, so the encoding's rules exist in exactly one
place, and let every existing reader keep doing what it already does.

**Canonical example**: `src/lib/people.js` (CSV-style quoting over a
comma-separated string, with a quote opening a quoted value only at the start
of one, so pre-existing values parse unchanged) and `src/lib/recurrence.js`
(a compact rule grammar inside a field that previously held a single keyword,
with the old keywords still parsing to their original meaning).

**When to use**: any schema change where a consumer is a *copy of logic* living
somewhere you cannot redeploy in the same commit. The test is not "is an object
cleaner?" — it is "what does changing the shape cost, and who pays it?" Here an
array would have rendered without spaces and, because an empty array is truthy,
printed a dangling "with " on every unassigned item; fixing that meant a manual
workflow re-import with every credential re-attached and a timezone re-pinned,
on a scheduled job whose failure mode is silence.

**Why it is not just laziness**: the encoding often has to solve the ambiguity
anyway. Joining names with `", "` when a name itself contains `", "` is
unreadable in the rendered output no matter how it was stored, so quoting had
to exist at the display layer regardless — which made keeping the string the
cheaper *and* the more correct option, not a compromise between the two.

**Trade-off**: encodings carry their own failure mode — a delimiter appearing
inside a value. Pay for that once, in a tested parser, rather than letting
heuristics spread across the readers. And say so in the code: the reason a
field looks under-modelled must be written down at its definition, naming the
consumer that constrains it. Otherwise the next contributor reads an
under-modelled field, "tidies it up" into an object, and breaks a consumer that
has no tests in this repo to catch it — the constraint is invisible from the
code that would be changed.

---

## Pattern: A drift guard covers a pair AND a field — enumerate the grid, because partial coverage reads as full coverage

**Used in**: Task Command, where the department list and the absence table each
exist in three copies — `src/constants.js` (the definition), `api/_lib/classify.js`
(a serverless function must not import from `src/`), and an n8n Code node (which
cannot import at all). Drift was already treated as a test failure and the suite
was green, yet two cells of the grid had never been checked: the api copy of the
*department list* was compared to nothing (only the workflow copy was), and the
absence table's `counts` flags were compared workflow-to-api but never to the
app.

**Shape**: when one table is duplicated across boundaries that cannot import
each other, the guards you need are not one per copy. They are one per **(pair
of copies × field that matters)**. Write that grid down and mark each cell
covered or not. Do it from the *table's* side, not the test file's — the
existing tests are organised by consumer (`test/n8n-brief.mjs`), so reading them
tells you which workflow is checked, never which cell of the table is.

**Why partial coverage is worse than none**: a passing check is named after the
table (`absence types match across app, api and workflow`), so it reads as
though the table is covered. Nobody re-derives which fields that sentence
actually compared. Values were checked; `counts` was not — and `counts` is the
whole reason the table has rows rather than being a list of strings, because it
is what separates "absent" from "present but restricted". A guard that covers
the cheap cell and skips the load-bearing one buys confidence without buying
safety.

**Also assert order when order is rendered.** Both copies were iterated to lay
out sections of an email, so equal-membership-different-order is a silent
behaviour change with nothing to catch it. `deepStrictEqual` on the arrays, not
a set comparison, whenever a consumer iterates rather than looks up.

**How to find the gaps**: grep for the constant's name across the whole repo
rather than reading the test files, then ask of each hit whether anything
compares it to the definition. The copies announce themselves in comments
(`Mirrors ABSENCE_TYPES in src/constants.js`) — a comment claiming a mirror with
no test behind it is the exact signature of an uncovered cell.

**Prove each new guard the usual way**: break one cell at a time and confirm the
matching check fails and only that one — see *Prove a new test discriminates by
breaking the implementation, not by deleting it*, above. Three mutations here
(drop a department, flip one `counts`, reorder two rows) each isolated one
check.

**Trade-off**: the grid grows as copies × fields, and not every field deserves a
cell — display-only fields (colours, short labels, lead times) legitimately live
in only one copy. Cover the fields a consumer *branches on*. The question to ask
of each field is not "is it duplicated?" but "if these two disagreed, would
anything raise?"

---

## Status

## Pattern: A suppression record's lifetime is set by what it suppresses, not by the queue that delivered it

**Recorded on one use**, against this file's own second-use bar, because the
failure it prevents is documented with a root cause (`LL-0105`) and is invisible
until it has already happened several times. Treat it as provisional until a
second use confirms the shape.

**Used in**: Task Command's `dismissedSources` — the sourceIds of deleted
calendar imports, held in the sync reducer beside the tombstone queue and
deliberately never drained, so a meeting deleted on one device is not
re-imported on the next calendar sync.

**Shape**: when an operation both *delivers* something and *suppresses*
something, two records are in play and they do not expire together. The
delivery record is transient — it exists until the other side acknowledges,
and draining it on acknowledgement is correct. The suppression record is
durable: what it protects against is not the un-acknowledged send but the
recreation, and the thing that recreates is usually a different component on a
different schedule. Keep them as separate collections with separate lifetimes,
and write down at the definition which is which.

**Canonical example**: `src/state/reducer.js` — `deletes` drains on `FLUSH_OK`
once the server confirms the tombstone; `dismissedSources`, populated in the
same `REMOVE` action, has no drain path at all. A test asserts exactly that
pairing, because the natural instinct is to clear both on the same signal.

**When to use**: any delete that a creator can undo — importers and ingestion
jobs, unsubscribe and blocklist handling, idempotency keys, dead-letter
suppression, mirroring with a delete log. The question that surfaces it: *after
this deletion is fully acknowledged, what would stop the record coming back?*
If the answer is "nothing, because the thing that made it will make it again",
the suppression record has to outlive the acknowledgement.

**Why it is not just a leak**: unbounded growth is the deliberate trade. The
set grows with lifetime deletions rather than live records, which for a
personal tracker is a few hundred strings; bounding it by time would
reintroduce the bug for any copy that was offline longer than the window. If
the size ever genuinely matters, the bound has to preserve the correctness
property rather than trade it away — see `docs/standards/data-lifecycle.md`,
where that choice is left open rather than decided.

**Trade-off**: a permanent suppression is permanent from the user's side too.
In Task Command a deleted calendar import can never be re-imported; it has to
be re-entered by hand. That is the right default for a deletion the user made
deliberately, but it must be a stated consequence rather than an emergent one,
and it argues for making the suppression *narrow* — keyed per occurrence rather
than per series, so deleting one week of a recurring meeting does not silently
suppress every week after it.

---

_Last reviewed: 2026-08-19, after adding the suppression-record-lifetime
pattern — recorded on a single use, against the second-use bar at the top of
this file, and marked provisional in the entry itself (from the Task Command
session where a deleted calendar meeting kept re-importing; see `LL-0105`).
Previously reviewed 2026-08-19, after extending the discriminating-test pattern to
cover self-initiated defensive code (a `hasOwnProperty` guard survived its
mutant because nothing tested it at all, unlike the three mutants beside it),
and after adding the drift-guard-grid pattern (from a
Task Command housekeeping session: the suite was green and the table was
"guarded", but two cells of the copies-by-fields grid had never been compared to
anything). Previously reviewed 2026-08-18, after adding the extend-the-encoding-not-the-type
pattern (from a Task Command session where a live n8n Code node read a field
directly, freezing its type; see `LL-0103` for the recurrence bug found in the
same session). Previously reviewed 2026-08-15, after cross-referencing
LL-0091 into the extract-after-the-second-consumer pattern — the same pair of
scripts later
showed the cost of leaving the extraction undone once a safety fix landed in
only one of them. Previously reviewed 2026-08-14, after adding the
freeze-and-unfreeze and extract-after-the-second-consumer patterns (from a
subagent-driven branch that added a self-hosted vector store beside a
production-proven upgrade script).
Previously reviewed 2026-08-03, after adding the credential-handoff and
ask-before-relaying-infeasible-data patterns (from a domain/Clerk production
cutover session that also closed out a 200-item bank review). Previously
reviewed 2026-07-27, when the discriminating-test pattern was added (from a
subagent-driven-development session building the last of five item types).
Earlier: 2026-07-25 (worktree post-dispatch verification, stalled-subagent
diagnosis) and 2026-07-01 (repository bootstrap) — the bootstrap note about
this file growing as concrete code patterns emerge in `src/` still applies,
no `src/` patterns yet._
