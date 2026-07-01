# Playbook: Bug Fix

Use this procedure whenever fixing a reported or discovered bug.

## Steps

1. **Reproduce** the bug and write a failing test that captures it
   (`docs/standards/testing.md`).
2. **Check `memory/lessons-learned.md`** for a related prior entry —
   this may already be a documented class of bug with a known preventive
   rule that wasn't followed, or a genuinely new class.
3. **Identify the root cause**, not just the symptom. Ask "why did this
   happen" at least until you reach a cause that, if fixed, would have
   prevented the bug entirely (not just this instance of it).
4. **Fix the root cause.** If a quick patch is applied under time pressure,
   explicitly note the deferred root-cause fix in `memory/roadmap.md` or
   `memory/future-ideas.md`.
5. **Confirm the test now passes**, and add any additional edge-case tests
   the root cause analysis surfaced.
6. **Record the lesson** in `memory/lessons-learned.md` per `CLAUDE.md`
   §15: Root Cause, Why It Happened, Solution, Preventive Rule, Similar
   Situations.
7. **Add the preventive rule** to the relevant `docs/standards/` document
   or review checklist if it's generally applicable, not just specific to
   this one bug.
8. **Summarize and propose a commit message**, then follow the Git
   Workflow in `CLAUDE.md` §10 (explicit approval before commit/push).

## Related Documents

- `CLAUDE.md` §15 — Learning System
- `docs/standards/testing.md`
- `memory/lessons-learned.md`
