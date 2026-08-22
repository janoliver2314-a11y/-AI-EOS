# Standard: Automation

## Purpose

Define when a recurring or event-driven process belongs in the shared n8n
instance versus in application code, and require that existing workflow
templates are checked before any automation is built from scratch. This
standard exists because automations were being hand-built per project while
a library of thousands of proven templates sat unused, and because the
workflows that did exist lived only inside n8n's database with no
recoverable copy.

## Scope

Applies to every project, current and future, whenever a process is
described as "when X happens, do Y" or "every <interval>, do Z" — email
and calendar handling, notifications, scheduled checks, webhook intake,
data sync between services, and light AI steps (classify, summarize,
draft). Core product logic that runs inside a user request is out of
scope: that is application code and stays in the project repository.

## Goals

- Stop rebuilding integrations that already exist as templates.
- Keep product code free of plumbing that has no business being there.
- Make every workflow recoverable and reviewable, not trapped in one
  database on one machine.
- Give future projects a single, known place to look for automation.

## Rules

1. **Plumbing goes to n8n; product goes to code.** If the job is moving
   information between tools, reacting to an external event, or running on
   a schedule, default to an n8n workflow in the shared instance
   (`lnc-n8n`, http://localhost:5678). If the job sits in the request path
   a user is waiting on, or carries logic that needs unit tests and typed
   contracts, write it in the project.
2. **Search the template library before building.** Before creating any
   workflow, search https://n8n.io/workflows (11,000+ community templates)
   for the service pair or pattern involved. Record what was searched and
   what was chosen — template name and URL, or "none suitable" with a
   one-line reason — in the project's `docs/` or the workflow's description
   field. Building from scratch without that note is a review finding.
3. **Adapt, do not trust.** An imported template is a starting point.
   Replace its credentials with the project's own, remove nodes the
   project does not need, and read every Code node before activating —
   community templates are unaudited.
4. **Back up after every change.** After creating or editing a workflow,
   run `~/Claude/Automation/Workflows/n8n/backup.sh`. The JSON exports in
   `~/Claude/Automation/Workflows/n8n/workflows/` are the reviewable,
   recoverable copy; n8n's database is the live copy, never the only one.
5. **No secrets in exports.** Workflows reference credentials by ID only.
   A secret inside a workflow JSON (in a Code node, HTTP header, or
   expression) is a violation of rule 5.5 of the operating manual and
   must be moved into an n8n credential before the next backup.
6. **Never touch n8n's database directly.** Use the n8n CLI inside the
   container, the REST API, or the MCP server. Opening the SQLite file
   from the host has corrupted state before.
7. **Name workflows by project prefix.** `LNC - …`, `NCLEX - …`,
   `Task Command — …`. One instance serves every project; the prefix is
   the only thing that keeps them sortable.

## Design Decisions

- **One shared instance, not one per project.** A single n8n keeps
  credentials, backups, and the MCP connection in one place. Project
  isolation comes from naming and from per-project credentials, which has
  been sufficient at the current scale. Revisit if two projects need
  conflicting n8n versions or if a project must be handed off entirely.
- **Template search is mandatory, template use is not.** The rule is to
  look, then decide. A template that is 80% right and adapted in ten
  minutes beats a from-scratch build; a template that is 30% right does
  not, and the note explaining why is enough.
- **Backups are committed exports, not database snapshots.** A JSON per
  workflow diffs cleanly in git and can be restored one at a time. A
  database dump cannot.

## Best Practices

- Search the library with the service names first ("Supabase Clerk
  webhook"), then with the intent ("onboarding email sequence").
- Prefer templates with a recent update date and a visible node count
  that matches the complexity you expect; a 40-node template for a
  3-step task is hiding something.
- Keep a workflow to one responsibility. Chain workflows with the
  Execute Workflow node rather than growing one into a monolith.
- Put the template provenance note in the workflow's own description so
  it survives export.

## Examples

Good — a new-signup welcome flow for a product:

1. Search "Clerk webhook Resend welcome email" in the template library.
2. Import the closest match, rename it `NCLEX - Welcome Sequence`, swap in
   the project's Resend credential, delete the unused Slack branch.
3. Write in the description: "Adapted from template #4821 (Clerk → Resend
   onboarding), removed Slack notify, 2026-08-22."
4. Activate, run `backup.sh`, commit the export.

Bad — the same flow written as a Next.js API route with a `setTimeout`
queue, because "it's only three emails." It is now product code that
needs tests, a deploy, and a cron provider, for a job that was plumbing.

## Common Mistakes

- Building the workflow first and searching templates afterwards "to
  compare." The search is the first step, not a validation step.
- Editing a workflow in the n8n editor and forgetting the backup; the
  export in git is now stale and the next restore loses work.
- Pasting an API key into an HTTP Request node header instead of creating
  a credential, then exporting it into the backup folder.
- Putting scoring, grading, or any user-visible calculation into n8n
  because it was quick. That is product logic; it belongs in code with
  tests.

## Future Improvements

- A pre-commit hook in `~/Claude/Automation` that fails if `backup.sh` has
  not run since the n8n database was last modified.
- A per-project `docs/automation.md` template listing the project's
  workflows, their triggers, and their template provenance.

## Related Documents

- `CLAUDE.md` §5 (summary rule 9) and §5.5 (no committed secrets)
- `docs/standards/dependency-management.md` — the same "adopt
  deliberately" posture applied to packages
- `docs/standards/security.md`
- `~/Claude/Automation/Workflows/README.md` — backup and restore procedure
