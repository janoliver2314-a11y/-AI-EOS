# Playbook: Proposing a New or Changed Standard

Use this procedure when a recurring situation reveals the need for a new
engineering standard, or when an existing standard needs to change.

## Steps

1. **Confirm it's actually missing or wrong.** Check `docs/standards/` and
   `CLAUDE.md` to make sure the rule doesn't already exist somewhere.
2. **Check for prior related lessons** in `memory/lessons-learned.md` — a
   new standard is often the generalized preventive rule from a specific
   bug.
3. **Draft the standard** following the skeleton in
   `docs/standards/documentation.md` (Purpose, Scope, Goals, Design
   Decisions, Best Practices, Examples, Common Mistakes, Future
   Improvements, Related Documents).
4. **Update the index** in `docs/volumes/06-engineering-standards.md`'s
   Standards Set table.
5. **Cross-link** from `CLAUDE.md` if the rule is broadly applicable enough
   to warrant a one-line summary there.
6. **If changing an existing standard**, update it in place — do not
   create a competing document — and note the change in `CHANGELOG.md`.
7. **Summarize and propose a commit message**, then follow the Git
   Workflow in `CLAUDE.md` §10.

## Related Documents

- `docs/standards/documentation.md`
- `docs/volumes/06-engineering-standards.md`
- `CLAUDE.md` §6
