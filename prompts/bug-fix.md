# Prompt: Bug Fix

Use this prompt (adapted as needed) to drive an AI session through fixing a
bug in compliance with AI-EOS's process.

---

> Read `CLAUDE.md` and follow `docs/playbooks/bug-fix.md` to fix the
> following bug: <describe the bug, including reproduction steps and
> expected vs. actual behavior>.
>
> Before writing a fix: check `memory/lessons-learned.md` for related prior
> entries. Write a failing test that reproduces the bug before fixing it.
> After fixing, add a `memory/lessons-learned.md` entry per `CLAUDE.md`
> §15, and propose a commit message. Do not commit or push without
> explicit approval.

## Related Documents

- `docs/playbooks/bug-fix.md`
- `CLAUDE.md` §15
