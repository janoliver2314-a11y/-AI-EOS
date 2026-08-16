# Lessons Learned

Structured record of bugs and mistakes, per `CLAUDE.md` §15. Every entry
follows the same shape: Root Cause, Why It Happened, Solution, Preventive
Rule, Similar Situations. Search this file before writing new code in an
unfamiliar area — never knowingly repeat a documented mistake.

Entries are numbered `LL-NNNN`, sequential, never renumbered or deleted.

---

### LL-0001 — Untrusted webhook payloads must be authenticated before parsing

- **Root Cause**: A hypothetical/example webhook handler pattern parsed and
  acted on payload fields before verifying the request's signature,
  meaning an attacker-controlled payload could influence behavior before
  authenticity was established.
- **Why It Happened**: Signature verification is easy to treat as "one more
  step" and insert after the main logic is written, rather than designing
  it as the first gate.
- **Solution**: Established the rule that signature/authenticity
  verification must be the first operation in any external-input handler,
  before deserialization or field access.
- **Preventive Rule**: Added to `docs/standards/security.md` and
  `docs/volumes/07-security.md`: verify authenticity before parsing content
  for any externally-triggered handler (webhooks, callbacks, uploads).
- **Similar Situations**: Any future webhook receiver, OAuth callback, or
  file upload handler — check `docs/standards/security.md` before merging.

---

### LL-0002 — Supervisor-managed persistent services don't inherit shell-exported env vars

- **Root Cause**: A CLI tool's "install as a persistent service" flow
  (Headroom's `install apply`, preset `persistent-service`) builds the
  service's runtime environment from a fixed manifest dict baked into a
  launchd-style service definition, not by inheriting whatever the calling
  shell had exported. Exporting `HEADROOM_REQUIRE_RUST_CORE=false` before
  running the installer had zero effect on the resulting service.
- **Why It Happened**: Assumed all subprocess-spawning code paths inherit
  `os.environ` from the caller (true for a plain `Popen` with no explicit
  `env=`), without checking that this specific path serializes a fixed env
  dict into a service definition that starts fresh at every future boot,
  decoupled from whichever shell happened to run the installer.
- **Solution**: Confirmed inheritance actually works on the code path this
  tool uses day-to-day (a hook-invoked subprocess, not the persistent-service
  path), then set the variable in a persistent location any interactive
  shell — and anything it later spawns — picks up automatically (`~/.zshrc`).
- **Preventive Rule**: Before relying on an exported shell variable to
  configure a background/persistent service, check whether the installer
  builds that service's environment from a fixed dict (grep the tool's
  source for `env=` / `os.environ.copy()`) rather than assuming inheritance.
  If it's a supervisor-managed service (launchd, systemd, Windows service,
  Docker `--restart`), the env must be baked into the service definition
  itself or set via a documented tool flag — a shell export will not reach it.
- **Similar Situations**: Any tool offering a "run as a service"/"persistent
  daemon" install mode — check where that mode actually reads its
  environment before assuming normal shell inheritance.

### LL-0003 — A successful `pip`/`pipx install` doesn't guarantee an advertised native component is present

- **Root Cause**: Installed a Python package (`headroom-ai`) that advertises
  a compiled Rust extension for accelerated behavior. On a platform with no
  prebuilt wheel (Intel macOS), pip/pipx silently resolved a fallback
  `py3-none-any` (pure Python) wheel with exit code 0 — no warning the
  native extension was missing. The gap only surfaced later, as a runtime
  crash when the app tried to import the absent extension.
- **Why It Happened**: Treated "install succeeded" as equivalent to "full
  capability available," without checking the installed wheel's platform
  tag or importing the native module before relying on it.
- **Solution**: Confirmed the gap by inspecting the downloaded wheel
  filename (`*-py3-none-any.whl` = no compiled code) and by reading the
  package's own source for how it handles a missing native module, then
  used its documented degraded-mode opt-out rather than continuing to fight
  a from-source build that hit further unrelated platform gaps.
- **Preventive Rule**: After installing a package that advertises a
  compiled/native-accelerated component, check the installed wheel's
  platform tag (`py3-none-any` means no native code) or explicitly import
  the native module before assuming full functionality — especially on
  platforms (Intel macOS, uncommon Linux archs) vendors commonly
  deprioritize for prebuilt binaries.
- **Similar Situations**: Any Python package with a Rust/C/C++ extension
  (tokenizers, pydantic-core, ML-adjacent libraries) installed on a
  non-mainstream platform — verify before assuming.

### LL-0004 — Homebrew install can fail on root-owned leftover directories from a prior install

- **Root Cause**: A first `brew install <formula>` failed with a
  permissions error on `/usr/local/share/zsh/site-functions` (and its
  parent), owned by `root:wheel` from a previous, unrelated Homebrew
  installation on the same machine — even though Homebrew itself had just
  been freshly (re)installed.
- **Why It Happened**: Assumed a fresh Homebrew install implies a fully
  clean, correctly-owned `/usr/local` tree; didn't check for stale
  root-owned artifacts left behind by an earlier Homebrew setup.
- **Solution**: `sudo chown -R <user> /usr/local/share/zsh
  /usr/local/share/zsh/site-functions`, then retried — the install worked
  immediately after.
- **Preventive Rule**: When a Homebrew install fails with a "not writable
  by your user" error right after installing/reinstalling Homebrew, check
  ownership of the exact paths named in the error (`ls -ld <path>`) rather
  than assuming a broken installer — it's almost always leftover root
  ownership from an earlier Homebrew instance.
- **Similar Situations**: Any machine that previously had Homebrew
  installed via `sudo` or a different user account — expect ownership
  mismatches in shared prefix directories (`/usr/local/share`,
  `/usr/local/etc`, `/usr/local/var`) on reinstall.

### LL-0005 — Low-code workflow nodes' "simplified" output field names/casing must be verified from real execution data, not assumed

- **Root Cause**: An n8n workflow (LNC dashboard's auto-log workflow) read
  `msg["to"]` / `msg["from"]` (lowercase) to extract recipient/sender, but
  the Gmail Trigger node's simplified output actually keys these fields as
  `"To"` / `"From"` (capitalized). Every extraction silently returned an
  empty string, so none of the workflow's Sheet-writing branches (Sent/FU1/
  FU2/Replied) ever matched a row. The workflow could never have auto-logged
  a real send or reply since the day it was built.
- **Why It Happened**: The field names/casing were assumed rather than
  checked against actual execution output, and nothing forced a check —
  existing tracker rows were already populated by a one-off manual backfill,
  not by this trigger path, so the workflow's total non-functionality
  produced no visible symptom. It was only caught by deliberately inspecting
  execution data during unrelated testing (a manual test send that got
  labeled but never appeared in the tracker).
- **Solution**: Corrected all 4 label branches to read `"To"` / `"From"`
  matching the trigger node's actual output, verified against real execution
  data rather than documentation or adjacent code.
- **Preventive Rule**: When building or modifying a low-code workflow
  (n8n, Zapier, Make, etc.) against a node's "simplified"/flattened output,
  verify the actual field names and casing from real execution data (via the
  platform's execution history/API, or by running the node once and
  inspecting output) before writing matching/extraction logic against them.
  Do not assume field names from documentation, memory, or adjacent code
  that was itself never validated against a live run.
- **Similar Situations**: Any workflow-builder node advertised as
  "simplified output" (Gmail, Slack, Notion, Airtable triggers/actions in
  n8n/Zapier/Make) — these simplification layers are undocumented or
  under-documented and can differ from the underlying API's field casing.
  Also applies to any integration code copied from one node/branch to
  another without re-verifying the new node's actual output shape.

### LL-0006 — n8n Code node execution mode and HTTP node concurrency are not what they look like

- **Root Cause**: Two related, silent low-code execution-model bugs, found
  while building an n8n workflow that discovers records via an external API
  and writes new rows into a Google Sheet:
  1. A Code node with no explicit `mode` set defaults to "Run Once for All
     Items," not "once per item." Code written assuming
     `$input.first().json` refers to "the current item" (per-item
     semantics) instead silently only ever processes the *first* item of
     the whole batch on every logical iteration, discarding the rest with
     no error — the node still reports success.
  2. An HTTP Request node given multiple input items fires them **in
     parallel by default**, not sequentially. When those parallel calls
     each read-then-write a shared resource (here: "find the next blank
     row, then write it"), they race: all read the same stale state, and
     only the last writer's data survives. This produced actual data loss
     — 13 of 14 genuinely discovered records were silently overwritten in
     one run, with the workflow still reporting success. The node's
     "batching" option (`batchSize: 1`) was tried as a fix and did **not**
     resolve the race in practice — confirmed empirically, not assumed.
- **Why It Happened**: Both defaults are internally consistent with how
  the platform is documented, but contradict the mental model a developer
  coming from ordinary sequential/per-item scripting brings by default.
  Nothing about the node's configuration UI or a quick glance at the code
  makes the batch-vs-per-item distinction, or the parallel dispatch,
  obvious — it only surfaces by inspecting real execution data
  (item counts per node run) after a live test.
- **Solution**: For per-item logic, set the Code node's mode explicitly to
  "Run Once for Each Item" and use `$json`/`$input.item.json` (not
  `$input.first()`) to reference the current item. For a group of writes
  that must land in contiguous/shared state (e.g., appending N new rows),
  eliminate the race structurally instead of trying to force serial
  HTTP dispatch: compute every write's target position once, up front,
  from a single read, then issue **one atomic batch write** covering all
  of them, rather than N separate calls that each independently resolve
  "the next" position.
- **Preventive Rule**: Before trusting any low-code (n8n/Zapier/Make)
  multi-item logic, verify empirically via the platform's execution
  history/API — check how many times each node actually ran and how many
  items it actually saw — rather than assuming batch-vs-per-item mode or
  request concurrency from the node's visual layout. Never let N
  concurrently-dispatched steps each independently read-then-write a
  shared/contiguous resource (a "next available slot," a running counter,
  a next-row scan); collapse them into one atomic operation instead.
- **Similar Situations**: Any n8n/Zapier/Make workflow with a Code/Function
  step processing a fan-out of items (search results, API pagination,
  webhook batch payloads) — verify the node's actual per-run item count
  before trusting `.first()`/`.item()`-style item access. Any workflow step
  that calls another webhook/workflow N times to each "find and claim" a
  slot in a shared spreadsheet, queue position, or counter — this is a
  race by default in any platform that parallelizes multi-item HTTP calls,
  not specific to n8n or Google Sheets.

### LL-0007 — Project bootstrap template shipped a tracked runtime-telemetry file, committed into every new project

- **Root Cause**: `Workspace/Bootstrap/template/` (the source `create-project.sh`
  copies into every new `Projects/<name>/`) included a pre-existing
  `.claude-flow/data/pending-insights.jsonl` file with no matching
  `.gitignore` entry. The template's own `.gitignore` covered `node_modules/`,
  build output, `.env`, and `.DS_Store`, but not tooling-runtime state.
  `create-project.sh`'s initial `git add -A && git commit` therefore committed
  this file into every bootstrapped project from its very first commit, and a
  later broad `git add -A` in one such project (NCLEX AI Platform) nearly
  staged further session-noise appended to it before being caught by
  inspecting `git diff --cached` prior to committing.
- **Why It Happened**: The template's `.gitignore` was written for
  general-purpose build/dependency artifacts and never audited against what
  the local Ruflo/claude-flow tooling itself writes into a project directory
  at runtime. A tool-local data directory looks like ordinary project content
  to `git add -A` unless explicitly excluded.
- **Solution**: Added `.claude-flow/` to `Workspace/Bootstrap/template/.gitignore`
  (fixes every future bootstrap) and to the already-bootstrapped NCLEX AI
  Platform project's `.gitignore`, with `git rm --cached` to untrack the file
  that had already been committed there.
- **Preventive Rule**: Before trusting a broad `git add -A` (especially a
  project's first commit, or any bootstrap/scaffold script that automates
  one), run `git status`/`git diff --cached` and check every staged path
  against what local tooling writes at runtime (`.claude-flow/`, similar
  agent/framework state dirs) — don't assume a scaffold template's
  `.gitignore` already accounts for tool-local state just because it covers
  common build artifacts.
- **Similar Situations**: Any new project bootstrapped before this fix landed
  (check `git ls-files | grep claude-flow` in each); any future addition of a
  new local tool/agent framework that writes its own state directory into
  project roots — audit `Workspace/Bootstrap/template/.gitignore` when that
  happens rather than waiting to catch it per-project.

### LL-0008 — TDD RED runs execute the unguarded code path; denial tests aimed at real data mutated the live dev DB

- **Root Cause**: An implementation plan's access-control denial tests (expect
  403 on every route) used a REAL seeded row id in their requests. During the
  TDD RED step — run deliberately before the gate exists — the requests hit
  the ungated handlers, which executed fully: a published question in the live
  local database got its content overwritten and its workflow stage moved.
  Nothing failed loudly; the damage surfaced only later, during browser
  verification, as a mysteriously un-published, mangled row in the review
  queue.
- **Why It Happened**: The plan author reasoned "the gate fires before handler
  logic, so the id doesn't need to exist" — true only AFTER the gate exists.
  The RED half of the TDD cycle runs the same requests against the pre-gate
  code, where they are ordinary, fully-privileged mutations. Integration tests
  against a live dev DB made those mutations real.
- **Solution**: Restored the DB from seed (`supabase db reset`), re-ran the
  full suite against the clean DB to prove the now-gated tests touch nothing,
  and changed the denial tests to a nonexistent id so even ungated handlers
  cap the blast radius at a 404 (NCLEX AI Platform commit c0a21c2).
- **Preventive Rule**: Negative/denial tests (401/403/404 assertions) must
  never aim mutating requests at real data ids — use ids that cannot resolve.
  When writing a plan's RED step, ask "what do these requests do against the
  CURRENT code, where the feature doesn't exist yet?", not just against the
  finished code.
- **Similar Situations**: Any auth/authz feature TDD'd with integration tests
  against a live datastore; rate-limit or validation tests whose requests
  would succeed destructively if the limiter/validator isn't wired yet; any
  test suite that borrows "convenient" seed rows as request targets for
  requests that are supposed to be rejected.

### LL-0009 — A degraded Docker Desktop needs a restart AND ancillary-service exclusion; `supabase start` fails the whole stack on one unhealthy container

- **Root Cause**: A `supabase db reset` aborted mid-run on a transient
  container health-check blip, leaving the local DB empty. Retrying while
  Docker Desktop was already resource-degraded made it worse: `supabase
  stop`/`start` began timing out and `docker exec` threw `setns` errors. Even
  a full Docker Desktop restart (`pkill -9` the `com.docker.*` + `Docker
  Desktop.app` processes, then `open -a Docker`) did not restore a working
  stack, because `supabase start` tears the ENTIRE stack down if ANY single
  container is unhealthy — and the ancillary `postgres-meta` and `studio`
  containers reliably fail their health checks on a cold Docker boot, taking
  the healthy `db`/`rest`/`kong` down with them.
- **Why It Happened**: Two compounding wrong assumptions. (1) "Just re-run the
  reset" (a prior, milder lesson) — insufficient once Docker itself is
  degrading, not just flapping once. (2) `supabase start` is all-or-nothing on
  health: a slow-to-warm optional service (studio/pg_meta) is treated
  identically to a broken essential one, so the services you don't even need
  for the task can block the ones you do. After a hard `pkill`, Docker
  Desktop's VM can also take 10+ minutes (and sometimes a GUI-level restart)
  to become ready — the daemon socket accepts connections but `docker version`
  hangs until the Linux VM finishes booting, which reads as "still wedged."
- **Solution**: Restart Docker Desktop, wait (patiently, or restart it again
  from the menu bar) until `docker version` returns a server version, then
  start Supabase with the non-essential services excluded:
  `supabase start -x studio,postgres-meta,imgproxy,edge-runtime,logflare,vector,supavisor`
  (valid exclude names: edge-runtime, gotrue, imgproxy, kong, logflare,
  mailpit, postgres-meta, postgrest, realtime, storage-api, studio, supavisor,
  vector). That brings up just db+rest+kong — everything a
  reset/verify/review path needs — after which `supabase db reset` runs
  cleanly and the seed applies.
- **Preventive Rule**: For local-Supabase-on-Docker verification, don't fight
  a degraded stack service-by-service: restart Docker Desktop, then start
  Supabase with `-x` excluding the ancillary services and only run `db reset`
  against the minimal db+rest+kong set. Treat a hung `docker version` after a
  hard kill as "VM still booting," not "still broken" — give it minutes, or
  restart Docker Desktop from the GUI.
- **Similar Situations**: Any local dev stack whose orchestrator gates startup
  on all-container health (Supabase CLI, some docker-compose healthcheck
  setups) — exclude/omit the services the task doesn't need rather than
  waiting on their flaky health checks; any workflow that hard-kills Docker
  Desktop and then races its VM boot. Reinforces the NCLEX AI Platform
  Docker-wedge notes.

### LL-0010 — A once-only / idempotency guard must be reset on the failure path, or a transient error becomes a permanent dead-end

- **Root Cause**: A React "finish the exam" action (`endExam`) set a
  `endingRef` guard `true` on entry to prevent a double-fire (two call sites
  — last-answer submit and the countdown's expiry — could race), but the
  `catch` block never reset it. When the finishing network calls
  (`completeSession` + `getSession`) failed transiently, the guard stayed
  `true`, so every later retry path (the timer's next tick, a re-submit)
  silently no-opped. Compounding it, the error banner was rendered only in
  the setup phase, so the failure was invisible while the learner sat on the
  last question — a permanent in-page dead-end from a recoverable error
  (server state was intact; UX only).
- **Why It Happened**: The guard was added to fix a concurrency bug (double
  submit) and reasoned about only the success path ("set true, then move to
  the summary"). The failure path — where the guard must be released so the
  operation can be attempted again — wasn't considered. Separately, error
  *state* was set (`setError`) but the error was only *rendered* in a
  different UI phase than the one the failing action runs in.
- **Solution**: Reset the guard in the `catch` (`endingRef.current = false`),
  clear stale error state at entry so a successful retry doesn't flash the
  old message, and render the error + an explicit retry control in the phase
  where the failing action lives — not only where the flow started (NCLEX AI
  Platform commit 3dac629).
- **Preventive Rule**: Any guard/latch/flag that gates an async action
  (idempotency keys, `isSubmitting`, once-only `firedRef`/`endingRef`,
  distributed locks) must be released on EVERY exit path, including the error
  path — otherwise a transient failure converts into a permanent stuck state.
  And surface an action's error (with a retry affordance) in the same UI
  context where the user triggered it; setting error state that only renders
  elsewhere is equivalent to swallowing it.
- **Similar Situations**: Optimistic-UI mutations with a "sending" latch; any
  "submit once" button ref; server-side idempotency-key handlers that mark a
  key consumed before the operation succeeds; a lock/semaphore acquired in a
  `try` without release in `finally`/`catch`; a state machine whose terminal
  transition has no path back out when the terminal action itself fails.

### LL-0011 — A port that answers is not your build: a stale dev server silently masked a new feature during verification

- **Root Cause**: During browser verification of a just-built API feature, a
  dev server from a *previous* session (started hours earlier, running
  pre-feature code) was still bound to the target port. The freshly launched
  server hit `EADDRINUSE` and exited — but its log printed "Application
  startup complete" *before* the bind error, and the port's health endpoint
  returned 200 (from the stale process). Verification proceeded against the
  old build: the new response field was silently absent and the UI rendered
  `NaN` where arithmetic used it.
- **Why It Happened**: "The port responds" was treated as "my server is
  running." Backgrounded dev servers outlive the session that started them;
  uvicorn's startup banner precedes the bind failure, so a quick log glance
  looks healthy; and a missing JSON field degrades silently in JS
  (`total - undefined === NaN`) instead of failing loudly.
- **Solution**: `lsof -nP -iTCP:8000 -sTCP:LISTEN` exposed the stale PID
  (start time hours before the feature existed); kill it, relaunch, confirm
  the log shows a successful bind, re-verify — field present, UI correct
  (NCLEX AI Platform, adaptive-selection verification, 2026-07-15).
- **Preventive Rule**: Before browser/e2e verification, prove the process
  answering the port is the build you just wrote: (1) read the new server's
  log far enough to confirm the *bind* succeeded (no `EADDRINUSE`), or
  (2) `lsof` the port and match the PID against the process just launched,
  or (3) hit an endpoint/field marker only the new build serves. A 200 from
  the port is evidence that *a* server is running, not that *yours* is.
- **Similar Situations**: A forgotten `next dev` serving yesterday's bundles
  on :3000; a Docker container publishing the same port as a local process;
  two checkouts of the same repo each starting "the" backend; CI e2e jobs
  hitting a leftover server from a previous job on a shared runner; hot
  reload silently dead so edits never reach the running process.

### LL-0012 — A Clerk *development* instance on a deployed (non-localhost) domain 404s to non-browser clients; don't diagnose the deployed frontend with curl

- **Root Cause**: During production-deploy smoke tests, `curl` and
  non-JS fetches of a live Next.js + Clerk frontend returned `404` on every
  route (`/`, `/practice`), while a real browser loaded the app and
  redirected cleanly to Clerk sign-in. The deployment was using a Clerk
  **development** instance (`pk_test_`/`sk_test_` on the shared
  `*.clerk.accounts.dev` domain), chosen because a true production instance
  (`pk_live_`) needs a custom domain + DNS the app didn't have yet.
- **Why It Happened**: Clerk dev instances authenticate via a "dev browser"
  handshake (a `__clerk_db_jwt` established through a redirect that only a
  real, JS-executing browser completes). `clerkMiddleware` calling
  `auth.protect()` on a signed-out request with no dev-browser token fails
  closed by *rewriting to `/404`* rather than redirecting — the response
  headers say exactly this: `x-clerk-auth-reason: protect-rewrite,
  dev-browser-missing`, `x-clerk-auth-status: signed-out`,
  `x-matched-path: /404`. curl can't do the handshake, so it always sees the
  404; the browser can, so it works. The 404 looked like a broken
  deploy/routing bug but was expected auth behavior.
- **Solution**: Verify a deployed Clerk-protected frontend **in a real
  browser**, not with curl. To confirm it's protection (not a real 404),
  read the response headers for `x-clerk-auth-reason` / `x-matched-path:
  /404`, or fetch through a JS-capable client. (NCLEX AI Platform, first
  production deploy, 2026-07-15 — a browser hit showed the Clerk hosted
  sign-in, labeled "Development mode", confirming the dev instance.)
- **Preventive Rule**: When a deployed, auth-gated page returns an
  unexpected status to `curl`/`wget`/`fetch`, check the auth middleware's
  response headers before treating it as a routing/deploy bug — an auth
  layer that rewrites-to-404 for unauthenticated (or handshake-incomplete)
  requests is indistinguishable from a real 404 by status code alone.
  Reserve command-line HTTP for endpoints that are genuinely public
  (health checks, JSON APIs that return their own 401); use a browser for
  anything a client-side auth SDK gates. And know that a Clerk **dev**
  instance is a stopgap on a deployed domain — plan the `pk_live_` +
  custom-domain swap for real launch.
- **Similar Situations**: Any auth middleware that rewrites unauthenticated
  requests to a 404/200 shell instead of a 401/302 (Clerk, some NextAuth
  setups); Vercel/Netlify deployment-protection walls returning 401/302 on
  preview URLs; a CDN/WAF serving a challenge page that scripted clients
  can't solve; SSR pages that render an empty shell to bots lacking a
  cookie the client JS would set.

### LL-0013 — `supabase status` omits the anon/service_role keys when `[auth] enabled = false`; CI must supply the local demo keys directly

- **Root Cause**: Wiring GitHub Actions CI for a FastAPI + Supabase app whose
  DB-integration tests hit PostgREST, the workflow tried to read the local
  stack's keys from `supabase status` to feed the test client. Every
  extraction returned nothing usable: `supabase status -o json | jq -r
  '.SERVICE_ROLE_KEY'` yielded `null` → the Supabase client raised
  `Invalid API key`; switching to `supabase status -o env --override-name
  auth.service_role_key=...` produced *empty* vars → `supabase_key is
  required`. Two red CI runs before the cause was found.
- **Why It Happened**: The project uses Clerk for auth, so
  `supabase/config.toml` sets `[auth] enabled = false`. With the auth
  (GoTrue) service disabled, `supabase status` **does not emit `ANON_KEY` /
  `SERVICE_ROLE_KEY` at all** — the JSON/env output only carries `API_URL`,
  `DB_URL`, `REST_URL`, etc. So both extraction attempts were reading fields
  that didn't exist. The keys themselves are still valid and required:
  PostgREST is running and verifies JWTs signed with the stack's JWT secret,
  independent of whether GoTrue runs.
- **Solution**: Set the keys **directly** in the workflow instead of
  extracting them. The local stack's anon/service_role keys are the
  well-known Supabase **demo** JWTs and are *deterministic* as long as
  `config.toml` does not override the JWT secret — so they can be committed
  as job `env` (`SUPABASE_URL: http://127.0.0.1:54321` plus the standard
  demo anon/service_role JWTs). They're safe to commit: they only
  authenticate against a throwaway local stack, never production. Confirmed
  the exact keys by starting the stack locally and curling PostgREST, and
  ran the full suite locally (136 passed) before pushing the fix. (NCLEX AI
  Platform, CI wiring, 2026-07-15.)
- **Preventive Rule**: Before scripting extraction of values from a tool's
  status/introspection output, confirm the fields actually exist under your
  config — a disabled service silently drops its fields rather than erroring.
  For local Supabase specifically: when auth is disabled, don't parse
  `supabase status` for keys; set the deterministic public demo keys (or a
  pinned JWT secret + derived keys) directly. And verify infra-dependent CI
  locally (bring the stack up, run the suite) before burning CI runs — two
  red runs guessing at CLI output is the tell you skipped this.
- **Similar Situations**: Any `status`/`describe`/`info` command whose output
  shape depends on which optional services/features are enabled (disabled
  module → missing field, not an error); parsing `kubectl`/`docker`/`gh`
  JSON for keys that only appear under certain configs; assuming an
  introspection endpoint always returns a field that is actually
  feature-gated; extracting secrets from tooling when the value is in fact a
  fixed, publishable default.



### LL-0014 — Background subagents park "waiting for a notification" that can never reach them; instruct foreground gates up front and nudge stalled agents to completion

- **Root Cause**: A subagent running as a background task has no notification
  channel of its own. When it launches long work (a ~5-min test suite, a
  build) as a *background* job inside its own session and then ends its turn
  "to wait for the completion notification," it stops permanently: the
  harness marks the agent completed at its last message, and the inner job's
  completion notice goes nowhere. The agent isn't crashed or slow — it has
  cleanly exited while believing it is waiting.
- **Why It Happened**: The wait-for-notification pattern is correct for the
  top-level controller session (which does get task notifications), and
  agents pattern-match to it. Long-running gates make backgrounding feel
  responsible. It happened four times in one session (Ember UI Phase-1
  build, NCLEX AI Platform, 2026-07-16/17) across different subagents —
  implementers on test runs and a fixer on a vitest run — even after the
  dispatch prompt warned against it once.
- **Recurrence (2026-07-22, same project, Ember Phase 3a subagent-driven-development
  branch)**: Happened again on a `PracticeView.tsx` retheme task, this time
  with a twist: the controller's own dispatch prompt for that task never
  included the foreground-gates instruction at all (despite this lesson
  already existing in memory) — proving the rule isn't self-enforcing; it
  has to be re-copied into every dispatch, every session, because a prior
  session's memory does not automatically shape a fresh dispatch prompt's
  wording unless the controller actively re-reads and reapplies it. The
  agent went idle ~4+ hours (previous occurrences were minutes). The first
  SendMessage nudge ("check if vitest finished, then report") *also*
  triggered the same stall one level down — the agent replied that it had
  its own `Monitor` watching the background process and would "report back
  when that completes," i.e. the nudge itself re-armed the wait-for-a-
  notification trap because it didn't forbid backgrounding. Only a second,
  blunter nudge ("stand down, do not commit, do not run anything further,
  the controller is taking over") actually stopped it. The controller broke
  the loop by verifying the diff directly in the shared worktree (`git
  diff`/`git status` — subagents and the controller share the same working
  tree, so an implementer's uncommitted edits are inspectable without
  needing the agent at all), running tsc/eslint/vitest itself in the
  foreground, and committing on the agent's behalf.
- **Solution**: Two-part controller protocol. (1) Every dispatch prompt for
  work involving long commands states explicitly: "Run gates in the
  FOREGROUND and wait (~N min is expected); never park waiting on background
  notifications — none will ever reach you; drive to completion or reply
  BLOCKED." (2) When a completion notification arrives whose result text
  says the agent is "waiting" for anything, treat it as a stall, not a
  result: immediately SendMessage the same agent — "no notification is
  coming; check the job's output directly or re-run in the foreground, then
  finish every remaining step without stopping." Resuming via SendMessage
  preserves the agent's context; re-dispatching fresh loses it and re-pays
  the setup cost. (3) **The nudge itself must repeat the no-backgrounding
  instruction explicitly** — a generic "check on it and report back" nudge
  can re-trigger the exact same stall if the agent still has its own
  Monitor/background job in flight. (4) If a second nudge doesn't produce a
  final status within one turn, stop negotiating with the stalled agent:
  inspect its edits directly via `git diff`/`git status` in the shared
  worktree, verify and run the gates yourself in the controller's own
  foreground shell, tell the agent explicitly to stand down and take no
  further action (so it can't race a late commit against yours), and write
  the report file on its behalf so the downstream task-reviewer dispatch
  still has its expected input file.
- **Preventive Rule**: A subagent's reply that ends in "waiting for X" is a
  stalled agent, not a status update — nudge it the moment you see it.
  Put the foreground-gates instruction in every dispatch that will run
  anything slower than ~1 minute — copy the sentence verbatim from this
  entry into the dispatch prompt itself, don't rely on the subagent or a
  future controller-self having read this file. Repeat it verbatim in every
  nudge too, not just the original dispatch. Treat "it's been stalled for
  hours" (not just minutes) as a stronger signal to stop nudging and
  recover directly rather than keep asking the agent to check on itself.
- **Similar Situations**: Any orchestrator/worker design where only the
  parent holds the event channel (CI orchestrators spawning jobs that poll
  APIs the child can't see); agents ending turns on "I'll wait for the
  webhook/callback"; humans in async workflows waiting on notifications that
  were routed to a different inbox; session-limit kills mid-agent (the
  sibling failure this session) — in both cases the recovery is the same:
  resume the same agent from its transcript with explicit "pick up from
  here, finish in one pass" instructions rather than starting over.

### LL-0015 — Google OAuth refresh tokens expire after 7 days while the consent screen is in "Testing" — reconnecting is a stopgap, publishing the app is the fix

- **Root Cause**: A self-hosted n8n instance authenticated to Google (Gmail,
  Sheets, Drive) via OAuth. The Google Cloud OAuth consent screen was left in
  **"Testing"** publishing status. Google unconditionally expires OAuth
  **refresh** tokens after 7 days for apps in Testing — so every credential
  sharing that OAuth app dies together roughly weekly, regardless of usage.
- **Why It Happened**: The OAuth app was set up just far enough to work
  ("Testing" is the default state after creating credentials) and never
  published, because it worked immediately and the 7-day timer is invisible
  until the tokens actually expire. The failure then presents as an
  *application* bug, not a credential one: the downstream consumer (a
  dashboard) showed `Object.entries requires that input parameter not be null
  or undefined` because the n8n webhook returned an empty (0-byte) 200 body
  when its upstream Google HTTP node failed — the real error
  (`Access could not be refreshed because the connected account has revoked
  access, the refresh token expired, or the account password or permissions
  changed`) was only visible in the container logs / execution data, not to
  the caller.
- **Solution**: Two layers. (1) *Immediate*: reconnect each affected Google
  credential in the n8n Credentials UI (manual OAuth sign-in). (2) *Durable*:
  Google Cloud Console → APIs & Services → OAuth consent screen → **Publish
  app** (Testing → In production). For a single-user internal app the
  "unverified app" warning is safe to ignore; publishing removes the 7-day
  refresh-token expiry entirely.
- **Preventive Rule**: When wiring any long-lived automation to Google OAuth,
  **publish the consent screen as part of setup** — never leave it in
  "Testing" for anything meant to run unattended. And when a downstream
  consumer of a webhook/automation returns empty-but-200, look at the
  automation's own execution logs for the real (often auth) error rather than
  debugging the consumer — an empty success body is a classic swallowed
  upstream failure.
- **Similar Situations**: Any unattended integration on Google OAuth (n8n,
  Zapier self-hosted, Make, custom scripts, cron jobs) in Testing status;
  more broadly, any OAuth provider with a "development/test" mode that caps
  token lifetime. Also any pipeline where a middle tier returns a 200 with
  no/empty body on upstream failure — treat empty-success as a failure
  signal, not a data-absent signal.

### LL-0016 — `Agent`'s `isolation: "worktree"` param creates a SECOND worktree even when the controller already set one up; only pass it when no shared worktree exists yet

- **Root Cause**: The controller had already entered a shared, dependency-installed
  worktree (via the native `EnterWorktree` tool) to execute an 11-task
  subagent-driven-development plan. When dispatching the first task's
  implementer subagent, the controller passed `isolation: "worktree"` on the
  `Agent` tool call out of habit (treating it as a generic "make this safe"
  flag). The `Agent` tool took that literally: it created an entirely
  separate worktree/branch, checked out from main's tip, and ran the
  subagent's Bash commands there — a directory with no `node_modules`
  installed. The subagent could still write files to the CONTROLLER's
  worktree via absolute paths (creating an orphaned test file there), but
  its actual shell commands (installs, test runs, commits) executed in the
  wrong, dependency-less checkout and got stuck.
- **Why It Happened**: `isolation: "worktree"` reads as "isolate this task,"
  which sounds like exactly what you want for a implementer subagent. But it
  means "create a NEW isolated workspace for this dispatch," which is
  redundant and actively harmful once the controller is already working
  inside a dedicated worktree for the whole plan — every subagent dispatched
  from inside that worktree already inherits its cwd and isolation from the
  controller's own working directory; it needs no isolation parameter of its
  own.
- **Solution**: Detected via the subagent's own transcript ("let me verify
  node_modules in the task's worktree" — wrong path). Told the stuck agent to
  stop and report exactly what it had touched (one orphaned file, zero
  commits). Removed the stray file from the correct worktree, confirmed via
  `git worktree list` that the phantom worktree had been auto-cleaned by the
  harness (it made no lasting changes). Re-dispatched every subsequent task's
  implementer WITHOUT `isolation`, adding an explicit "cd into the exact
  worktree path and verify branch name + `node_modules` presence before
  anything else" preamble as a second line of defense.
- **Preventive Rule**: Before adding `isolation: "worktree"` to any `Agent`
  dispatch, ask: is the controller's OWN cwd already inside a dedicated
  worktree for this unit of work? If yes, dispatch subagents with no
  isolation parameter at all — they inherit it. Only pass `isolation` when
  dispatching from a shared/default checkout that itself needs protecting
  from the subagent's changes.
- **Similar Situations**: Any tool with a "make this isolated/sandboxed" flag
  that actually means "create a new isolated environment" rather than
  "inherit the caller's isolation" — read the exact semantics once per tool,
  don't pattern-match from the flag's name. More generally: subagent-driven
  development inside a pre-established worktree should treat that worktree
  as the ambient environment for every dispatch in the plan, not something
  to re-request per task.

### LL-0017 — The Claude Code preview harness injects its own credentials into dev servers it manages by name, silently overriding the project's own `.env.local` for known auth-provider keys

- **Root Cause**: `preview_start({name: "<config>"})` (the Browser pane's
  named-server launcher, per `.claude/launch.json`) starts the dev server
  process itself and appears to pre-seed environment variables for common
  auth providers (confirmed for Clerk: `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY`)
  so that a fresh preview "just works" for sign-in without any project setup.
  Since `dotenv`-style loaders (including Next.js's) never override an
  already-set `process.env` value, this pre-seeded value silently wins over
  whatever the project's own `.env.local` specifies — even after editing
  `.env.local`, stopping the server, clearing `.next`, and restarting.
- **Why It Happened**: The project needed to be pointed at a *different*
  Clerk instance (a cloud/production-tied test instance) than whatever the
  harness defaults to, to browser-verify a feature branch against real cloud
  data. Every restart of the harness-managed server kept resolving to the
  harness's own Clerk instance (`engaged-opossum-28.accounts.dev`) instead of
  the one named in `.env.local` (`crisp-squirrel-45.accounts.dev`) — read at
  first as a caching bug (tried clearing `.next`, hard-reloading, fresh
  browser tabs), when the actual cause was an env variable set outside the
  file entirely, in the harness's own process-launch environment for that
  named config.
- **Solution**: Don't rely on `preview_start({name})` when a project's real
  env file needs to take effect. Instead: (1) start the dev server yourself
  as a plain background process (`nohup npm run dev -- -p <port> &`, same
  pattern already used for backend processes per LL-0002-style guidance —
  use a port other than the harness's default to avoid a collision), (2)
  open it with `preview_start({url: "http://localhost:<port>"})` instead of
  `{name}` — the `url` form just opens a browser tab against an
  already-running server and does not launch or manage the process, so no
  env injection happens. Confirmed this actually resolves to the project's
  configured Clerk instance.
- **Preventive Rule**: Use `preview_start({name})` for routine "does this
  page render" checks where default/harness-provided credentials are fine.
  The moment a task requires the app's OWN configured secrets (a different
  auth tenant, a specific API key, cloud vs. local backend) to be in effect,
  run the dev server as a plain background process and open it via
  `preview_start({url})` instead — never assume clearing caches or
  restarting a harness-managed server will make a `.env.local` change take
  effect for a variable the harness itself seeds.
- **Similar Situations**: Any managed/sandboxed dev-server launcher that
  pre-populates "convenience" credentials for common integrations (auth
  providers, payment sandboxes, analytics) — the same silent-precedence trap
  applies to any env var name the launcher recognizes, not just Clerk's.
  More generally: when an env change doesn't take effect after a full
  restart and cache clear, suspect an env var already set upstream of the
  file loader, not a caching problem — check `env | grep <VAR>` in the
  actual process's launch context if possible, or falsify the caching
  hypothesis by moving to a manifestly different process/port.

### LL-0018 — The Browser pane's `computer` synthetic `left_click` can land on the exact right DOM element and still not fire the page's event handlers; fall back to a native `.click()` via `javascript_tool`

- **Root Cause**: Clicking a button via `computer{action:"left_click", ref:
  ...}` sometimes produces no visible effect — no state change, no network
  request, no console error — even though `document.elementFromPoint()` at
  that exact coordinate confirms the click landed on the intended button and
  not an overlay. The synthetic input event the automation layer dispatches
  doesn't always propagate through to React's synthetic event system on this
  page (likely a pointer-event-sequence mismatch: React's delegated listeners
  can expect a specific pointerdown/mousedown/mouseup/click sequence that an
  automated single "click" doesn't always reproduce faithfully).
- **Why It Happened**: Repeated clicks on a "10 questions" button (by ref, by
  coordinate, after re-reading fresh refs) produced zero network requests
  three times in a row, ruling out stale-ref and coordinate-scaling
  explanations before landing on the actual cause. `elementFromPoint` at the
  click coordinate matching the target element proved the click reached the
  right pixel; only dispatching `element.click()` directly via JS confirmed
  the handler itself was fine and would fire immediately given a real click
  event.
- **Solution**: When a `computer` click produces no observable effect after
  confirming (via `elementFromPoint` or similar) that it targeted the right
  element, dispatch a native click instead:
  `javascript_tool` with `element.click()` on the located element. This
  bypasses the synthetic-input layer entirely and reliably triggers React's
  handlers. Used this as the primary interaction method for the rest of a
  multi-step browser verification (selecting answers, submitting, expanding
  `<details>`, clicking nested buttons) once the pattern was identified,
  rather than re-attempting `computer` clicks each time.
- **Preventive Rule**: If a `computer` click (or tap/press) doesn't produce
  the expected effect within ~1-2 seconds and the console/network show no
  errors, don't assume the app is broken — verify the click landed on the
  right element, then retry via `javascript_tool`'s native `.click()` before
  concluding there's a real bug. For a browser-verification pass with many
  repeated interactions, prefer `.click()` via JS from the start once this
  symptom has appeared once in the session.
- **Similar Situations**: Any browser-automation tool whose "click" is a
  synthesized OS/CDP-level input event rather than a DOM-level dispatch —
  the same silent-no-op risk applies to `left_click`, `double_click`, and
  potentially keyboard input on frameworks with non-trivial synthetic event
  systems (React, and similar). A native `element.click()` / `dispatchEvent`
  fallback via an in-page JS execution tool is the general workaround.

### LL-0019 — A freshly created git worktree (default `fresh` base ref) silently excludes local commits that haven't been pushed to origin

- **Root Cause**: The native `EnterWorktree` tool creates a new worktree
  branched from `origin/<default-branch>` by default (`worktree.baseRef:
  fresh`), not from the current local branch's HEAD. Two commits (a design
  spec + an implementation plan) had been committed to local `main` but not
  yet pushed to `origin/main` before the worktree was created to execute
  that very plan. The worktree's branch started one HEAD behind local
  `main`, silently missing exactly the two files the whole session was
  about to execute — a plan-extraction script failed with "no such plan
  file" because the plan simply wasn't there.
- **Why It Happened**: The habitual sequence (brainstorm → write spec →
  write plan → commit both → enter worktree → execute) had never previously
  hit this, because in prior sessions the preceding work had already been
  pushed, or the worktree was created before any new local-only commits
  existed. Nothing about "start a worktree" signals that its default base
  ref is the *remote* tracking branch rather than local HEAD — it's a
  reasonable-sounding but wrong assumption that a fresh worktree starts
  "from where I am now."
- **Solution**: Ran `git log --oneline main -3` from inside the worktree to
  confirm the local `main` ref was still visible (worktrees share the same
  object database, so unpushed local commits are reachable even though the
  worktree's own branch doesn't start from them), then `git rebase main`
  to fast-forward the worktree branch onto local `main`, which picked up
  both missing commits cleanly with no conflicts.
- **Preventive Rule**: Before creating a worktree (via `EnterWorktree` or
  `git worktree add`) to execute work whose prerequisites were just
  committed locally, either (a) push the current branch to origin first, or
  (b) immediately after entering the worktree, run `git log --oneline
  <original-branch> -3` and diff against the worktree's own `HEAD` — if the
  worktree is missing expected commits, `git rebase <original-branch>`
  before doing anything else. Don't assume a just-made local commit is
  automatically present in a newly created isolated workspace.
- **Similar Situations**: Any tool or workflow that creates an isolated
  workspace (a fresh container, a CI runner, a second git worktree, a cloud
  sandbox) whose default checkout point is a remote/published ref rather
  than the requesting session's actual local state — the same class of "my
  recent local-only work silently isn't there" surprise applies whenever
  isolation is implemented via "clone/branch from the canonical remote"
  rather than "snapshot my current state."

### LL-0020 — `supabase start`'s own exit status isn't a reliable signal for whether the app can actually run; check individual container health, not the CLI wrapper

- **Root Cause**: `supabase start` repeatedly reported failure and
  auto-tore-down the whole stack (`Stopping containers...`) because
  `supabase_pg_meta` (the Postgres-introspection sidecar that only powers
  Supabase Studio's admin UI) failed its health check — even while `db`,
  `kong` (the API gateway), and `rest` (PostgREST) were already up and
  healthy. The actual application (a FastAPI backend + Next.js frontend
  that only ever talk to `kong`→`rest`, never to `pg_meta`/Studio) had
  everything it needed the whole time; the CLI's own success/failure gate
  is stack-wide, not scoped to what a given caller actually needs.
- **Why It Happened**: Treated `supabase start`'s exit code as the source
  of truth for "is the local backend usable," rather than checking `docker
  ps --filter name=supabase` directly to see which specific containers were
  healthy. Each failed `supabase start` attempt tore the whole stack down
  again, needlessly repeating db/kong/rest's (successful, slower) startup
  work before the next retry.
- **Solution**: After a `supabase start` failure, ran `docker ps --filter
  "name=supabase" --format "table {{.Names}}\t{{.Status}}"` directly —
  confirmed `db`/`kong`/`rest`/`inbucket` were healthy despite the CLI's
  reported failure — and proceeded to start the app servers against them
  without waiting for `pg_meta`/`studio` to recover. (In this case,
  `pg_meta` also self-healed on a subsequent plain retry a couple of
  minutes later, so simply retrying works too — but isn't necessary to
  unblock app-level work.)
- **Preventive Rule**: When `supabase start` (or any multi-service
  `docker compose`-style stack launcher) reports failure, don't
  retry-and-wait blindly — check which specific containers/services are
  actually healthy via `docker ps`, and identify whether the failing
  service is on the critical path for the task at hand (here: an
  admin/introspection UI, not the data path). Only chase the launcher's own
  success if the task genuinely needs the failing service.
- **Similar Situations**: Any local dev stack composed of multiple
  containers/processes with an aggregate "all healthy or nothing" launcher
  (docker-compose, supabase CLI, firebase emulators, LocalStack) — the
  aggregate gate is usually stricter than what any single task actually
  requires. Generalizes to: when a multi-component launcher fails, check
  the component graph before assuming the whole thing is unusable.

---

### LL-0021 — An open-ended `vitest run` that "hangs" for many minutes was host resource contention (load average ~80x core count), not a code or test bug — and my own diagnostic process let it run unbounded and killed the wrong process while investigating

- **Root Cause**: Two distinct problems stacked. (1) I ran a full frontend
  test suite (`npx vitest run`) without any self-enforced timeout, using
  only the tool's own foreground call, which silently backgrounds a
  long-running command rather than killing it — so a genuinely stuck run
  could and did continue indefinitely with no automatic alert, until the
  user asked "are you sure this didn't freeze?" (it should never have taken
  a user prompt to raise that). (2) Once investigating, my first kill
  attempts (`kill $VPID` where `$VPID` was captured from `npx vitest run &`)
  killed the `npx` wrapper process, not the actual `node .../vitest` process
  it spawned — so every "killed and retried" attempt left the previous
  vitest process running as an orphan, and subsequent runs were silently
  contending with 1-2 leftover vitest processes for the same resources,
  making later attempts look like the same hang recurring. The underlying,
  real cause (once processes were cleaned up and reproduced cleanly) was
  host-level: `uptime` showed a load average of ~390-530 on a 6-core
  machine (65-90x normal) for over ten sustained minutes, with idle-looking
  CPU% but heavy disk I/O and active Spotlight indexing
  (`mdworker_shared`/`mdmclient`) — most likely a newly-`npm install`ed
  worktree's `node_modules` being indexed, compounded by (per this
  environment's own docs) the possibility of other concurrent Claude
  sessions sharing the same host. Vitest's worker pool (`pool: "threads"`,
  and separately confirmed with `pool: "forks"` too) failed identically
  with `[vitest-pool-runner]: Timeout waiting for worker to respond` for
  even a single trivial test file — the pool couldn't spawn and get an
  initial response from a worker in time, which is exactly what host
  starvation looks like, and looks superficially identical to "the test
  file is stuck."
- **Why It Happened**: I treated "run the full test suite" as a plain
  foreground command instead of a long/uncertain-duration command needing
  an explicit kill deadline, so nothing forced me to notice or report a
  stall on my own — the user had to ask. Then, mid-diagnosis, I killed
  processes by the `$!` of the shell that launched `npx`, not the actual
  node process `npx` execs into — an easy mistake since `npx` is
  transparent in normal use, but it means "kill the PID I started" is not
  the same as "kill the process actually doing the work." Finally, I didn't
  check `uptime`/host load as a first-class diagnostic step until well into
  the investigation — I bisected test *files* for a long time on the
  assumption the problem was in test content, when the very first symptom
  (hangs before printing even one result, for any file, in any pool mode)
  was already a strong signal to check host state immediately rather than
  the code.
- **Solution**: (1) Killed every orphaned `vitest`/`npx` process with
  `pkill -9 -f "vitest run"` (matches the real process by command line, not
  a captured PID). (2) Confirmed via `uptime` that load stayed pinned in
  the 380s-450s range for the full 10-minute observation window (not a
  brief spike), and via `top -l 1` that Spotlight indexing processes were
  active with heavy disk I/O despite idle CPU%. (3) Reported the honest
  state to the user instead of claiming the frontend-suite gate was clean:
  every individual test file already had a verified pass from its own
  task's implementer+reviewer, so code correctness wasn't in question —
  only the aggregate run was blocked, and was recorded as such in the
  session's progress ledger rather than silently retried into a false
  "done."
- **Preventive Rule**: Added to `CLAUDE.md` §16 (Error Prevention Rules):
  (a) every long/uncertain-duration shell command gets an explicit,
  self-enforcing kill deadline, never an open-ended foreground call left to
  "check on later"; (b) when a command that should take seconds is still
  running after minutes, check `uptime`/host load *before* assuming the
  code under test is broken — a load average many times the core count is
  the first thing to rule out, not the last; (c) when killing a process
  launched through a wrapper (`npx`, `npm exec`, a shell function), kill
  the real underlying process via `pkill -f <real binary>`, not the
  wrapper's `$!`, and verify via `ps`/`pgrep` that it's actually gone
  before retrying.
- **Similar Situations**: Any "the tests just hang" report on a fresh
  worktree/`npm install` (Spotlight/antivirus indexing a large new
  `node_modules` is a common, overlooked cause on macOS); any worker-pool-
  based test runner (Vitest, Jest with `workerThreads`, Playwright's
  parallel workers) reporting a timeout waiting for a worker to respond —
  treat that error message class as a host-resource-first diagnosis, not a
  test-content bisection, especially when it reproduces on a single trivial
  file; any diagnostic loop that repeatedly "kills and retries" a
  wrapper-spawned process without confirming the underlying process is
  actually dead each time; any shared/multi-tenant execution host where
  another session's load is invisible until `uptime` is checked directly.
  Complements `LL-0014` (background *subagents* silently stalling waiting
  on a notification that can't reach them) — this entry is the mirror case
  for the controller's own foreground commands: don't let anything, agent
  or shell command, run unbounded and unmonitored.

---

### LL-0022 — Read/Write/Edit-class tools resolve paths against a process's spawn cwd, not the shell's `cd` state — a git worktree's isolation is invisible to them

- **Root Cause**: Inside a git worktree (created via a native `EnterWorktree`
  tool or `git worktree add`), Bash's `cd` genuinely changes the shell's
  working directory, but file-editing tools (Read/Write/Edit, and any
  subagent's own copies of them) resolve relative — and even some
  absolute-looking — paths against the process's original spawn root, not
  the shell's current directory. Two independent incidents in one session
  confirmed this: (1) a dispatched implementer subagent committed its
  Task-2 work to the *main* checkout instead of the worktree, even though
  its dispatch prompt told it to `cd` into the worktree first; (2) the
  controller itself (not a subagent) edited a plan file using a
  worktree-prefixed absolute path that still silently resolved to the main
  checkout's copy of the same relative path, because the path was typed
  from memory rather than freshly derived from `pwd`.
- **Why It Happened**: `cd` is a shell built-in that only affects the shell
  process's own state; it has no effect on a tool call's internal path
  resolver, which was already primed with a root at process/session start.
  This is easy to miss because Bash commands issued right after `cd` behave
  exactly as expected, creating false confidence that the whole toolset is
  now "in" the worktree.
- **Solution**: For incident 1, cherry-picked the subagent's commit onto
  the worktree and hard-reset main to discard it. For incident 2, committed
  the correct edit on main, pushed, and fast-forward-merged it into the
  worktree.
- **Preventive Rule**: (a) Every subagent dispatch into a worktree gets a
  mandatory first instruction: run `pwd` and `git branch --show-current`,
  confirm both match the worktree before touching any file, and report
  back if they don't. (b) Every file path used in a dispatch prompt or a
  controller's own tool calls must be the full, freshly-derived
  worktree-absolute path (from that session's own `pwd`/`git rev-parse
  --show-toplevel`), never a path typed from memory or copied from an
  earlier main-checkout context. (c) After every subagent dispatch that is
  supposed to commit, independently verify with `git log` in *both* the
  worktree and the main checkout — never trust a subagent's self-reported
  "committed" without checking where.
- **Similar Situations**: Any multi-worktree or multi-checkout workflow
  (git worktrees, multiple sandboxed containers sharing a filesystem,
  parallel subagent fleets each meant to operate in a different directory)
  where isolation is enforced at the shell/process level but tool calls
  have their own, separately-primed notion of "current directory." Treat
  this as true for any harness until proven otherwise for that specific
  harness.

### LL-0023 — A migration that adds a CHECK constraint before backfilling existing rows passes on an empty dev-reset DB and fails against real production data

- **Root Cause**: A migration added a `CHECK` constraint on a column in the
  same transaction as the `ALTER TABLE ... ADD COLUMN`, then backfilled
  existing rows with an `UPDATE` afterward. Statement order inside the
  migration file was: add column → add constraint → backfill. Against a
  freshly reset local dev database (`supabase db reset`, or equivalent
  migrate-from-empty flows), this always passes because there are zero
  existing rows for the constraint to reject at the moment it's added.
  Against the real cloud/production database, which already had rows, the
  `ADD CONSTRAINT` step validated every existing row against the new
  constraint immediately and failed, because those rows hadn't been
  backfilled yet — the backfill statement never got a chance to run.
- **Why It Happened**: Local dev workflows built around "reset from
  scratch and replay every migration" never exercise the
  constraint-against-existing-data path that a real, already-populated
  database always exercises. A migration can look completely correct and
  pass every local test while being silently broken for the one
  environment — the real database — where migrations actually have
  consequences.
- **Solution**: Reordered the migration's statements: add column (nullable
  or with a safe default) → backfill existing rows via `UPDATE` → add the
  `CHECK` constraint last, once every row already satisfies it. Applied
  the corrected order directly to fix the live database, and also
  corrected the migration file itself so the same bug can't recur on the
  next fresh deploy or reset.
- **Preventive Rule**: For any migration that adds both a new column and a
  constraint referencing it: constraints always go *after* any backfill
  that populates the column for existing rows, never before. When only a
  local/empty-DB test is available, treat that pass as necessary but not
  sufficient — explicitly re-read the statement order and ask "would this
  survive running against a table with existing rows that violate the new
  constraint until backfilled?"
- **Similar Situations**: Any schema migration adding `NOT NULL`, a `CHECK`,
  or a `FOREIGN KEY` constraint to an existing table in any SQL database
  (Postgres, MySQL, SQLite) — the ordering rule is universal, not
  Postgres-specific. Especially relevant whenever local dev and production
  diverge in whether migrations are actually kept in sync (see the
  "migration gap" pattern below).

### LL-0024 — A test that calls a framework-routed function directly bypasses the framework's own response validation

- **Root Cause**: A fix changed the *shape* of what a function returned
  (from a flat list to a per-group dict) and was verified by a test that
  imported the function and called it directly — the test passed. That
  function's return value was also serialized through a web framework's
  declared response model (FastAPI + Pydantic in this case), which
  performs its own validation/serialization pass at the HTTP layer,
  independent of the function's own return type hints. The first real HTTP
  request through the endpoint threw a 500 response-validation error that
  the passing unit test never had any chance of catching, because calling
  the Python function directly skips the framework's request/response
  pipeline entirely.
- **Why It Happened**: A test that imports and calls a service function
  looks like it's testing "the real code," and for pure logic it is — but
  for any function whose return value flows through a framework boundary
  (a FastAPI `response_model`, a DRF serializer, a GraphQL resolver's
  declared type), the framework's own validation is part of the contract
  and isn't exercised unless the test goes through that boundary.
- **Solution**: Widened the declared response-model type to match the new
  return shape, and added a genuine HTTP-layer test (via the framework's
  test client hitting the real route) that reproduces the original 500 and
  now passes.
- **Preventive Rule**: Whenever a change alters the *shape* (not just the
  value) of data returned by any function whose return type is also bound
  to a framework response/serialization contract, the regression test for
  that change must go through the real request path (`TestClient`,
  equivalent HTTP test client, or an actual request), not just call the
  function directly. A service-level unit test proves the logic is right;
  it proves nothing about whether the API contract the framework enforces
  still holds.
- **Similar Situations**: Any typed API framework with declared
  request/response schemas (FastAPI + Pydantic, DRF serializers, GraphQL
  resolvers, gRPC service definitions, tRPC procedures) — a passing
  function-level test for a handler/resolver/service method is not
  evidence the framework's own serialization layer still accepts its
  output. Check this explicitly whenever a task's diff touches a function
  whose return type doubles as a framework contract.

### LL-0025 — Reading a containerized SQLite database from the macOS host corrupts the running app's connection

- **Root Cause**: Ran `sqlite3` on the host against an n8n database file
  that lives in a Docker bind mount (`database.sqlite` in WAL mode) while
  the container was actively using it. SQLite's WAL coordination relies on
  a shared-memory file (`-shm`) and POSIX advisory locks, and Docker
  Desktop's virtiofs file sharing cannot coordinate either across the
  host/guest kernel boundary. The running app's pooled connections started
  failing every query with `SQLITE_NOTADB: file is not a database`.
- **Why It Happened**: Read-only queries feel harmless — "I'm just
  SELECTing." But WAL-mode SQLite readers are not passive: they open the
  `-shm` file, take locks, and can trigger WAL checkpoints. Two kernels
  each maintaining their own view of those locks over virtiofs is
  undefined behavior, regardless of read vs. write intent.
- **Solution**: `docker restart <container>` — the app reopened the
  database cleanly (WAL replayed, nothing lost). Switched all further
  reads to the app's own HTTP API instead of the file.
- **Preventive Rule**: Never open a SQLite file with host tools while a
  container is using it through a bind mount — not even read-only. Read
  through the app's API (webhook, REST) or `docker exec` a query inside
  the container so exactly one kernel owns the locks. If host-side
  analysis is truly needed, copy the `.sqlite` + `-wal` + `-shm` files
  first and open the copy.
- **Similar Situations**: Any file-locking-dependent store accessed across
  a VM/host file-sharing boundary — SQLite under Docker Desktop
  (virtiofs/gRPC-FUSE), LMDB or DuckDB files in bind mounts, even two
  SQLite clients on an NFS mount. Also applies in reverse: a host app's
  SQLite DB should not be queried from inside a container via a mount.

### LL-0026 — A dev server launched from the wrong checkout mimics missing code exactly

- **Root Cause**: Browser-verifying a feature built in a git worktree
  against dev servers that were launched from the main checkout — the
  running process served pre-feature code, producing a
  `ResponseValidationError` that named exactly the union members the
  feature was supposed to have added, indistinguishable at first glance
  from "the union widening was never done."
- **Why It Happened**: The preview/launcher config was anchored at the
  main project root; the port responding looked like "the server is up."
  Nothing forced the check that the server's watch/root directory was the
  worktree. Third occurrence of the stale-server class in one project.
- **Solution**: The server's own startup log (uvicorn prints its watch
  directory) identified the wrong root in one line. Added launcher entries
  pinned to the worktree cwd; re-verified against those.
- **Preventive Rule**: Before browser/manual verification of worktree- or
  branch-isolated work, confirm the running server's root/watch directory
  IS the isolated checkout — read the startup log line; never infer from
  the port responding or the app rendering. Also check `git status` in the
  MAIN checkout after subagent work in a worktree: a subagent can write to
  both trees (found byte-identical strays this session).
- **Similar Situations**: Any hot-reload dev server + worktree/multi-clone
  workflow; docker-compose mounting the wrong checkout; a globally
  installed CLI shadowing the repo-local build; "works locally" because a
  stale process on the port predates the fix. The both-trees warning above
  recurred and has its own entry with the proven mechanism: see LL-0031.

### LL-0027 — Queries that grow with a table, hidden by local data lagging prod scale

- **Root Cause**: An API `count` parameter had a lower bound but no upper
  bound; an oversized value silently capped to "everything published," and
  the resulting id list was rendered by PostgREST into the query string —
  so request size grew with the TABLE, not the request, until the URI blew
  past the server limit (HTTP 414).
- **Why It Happened**: Local seed data lagged production scale (371 rows
  locally vs 572 in prod), so the full local suite stayed green while the
  failure was already reachable in production. The gap surfaced only when
  local data was backfilled to prod scale and four tests failed at once.
- **Solution**: Chunked the id-list fetch (bounded URI regardless of bank
  size), then bounded the input at the system boundary (`le=` a
  domain-meaningful maximum), each with regression tests proven to fail
  when neutered.
- **Preventive Rule**: (1) Keep local/test data representative of
  production scale, or add one test that runs at prod scale — a green
  suite against smaller data proves nothing about size-dependent failures.
  (2) Never build a request whose size scales with a table (id lists in
  query strings, giant IN clauses); chunk or POST. (3) Every numeric input
  at a system boundary gets an upper bound rooted in the domain.
- **Similar Situations**: `IN (...)` lists hitting max query length;
  HTTP-header/URL limits behind REST-over-query-string layers (PostgREST,
  OData); pagination-free "fetch all" endpoints that work until the table
  grows; batch APIs whose payload grows with a join.

### LL-0028 — "Fails before the fix" evidence is worthless unless the failure is behavioral

- **Root Cause**: A fix wave shipped regression tests plus the claim they
  failed pre-fix — but the evidence came from stashing the fixed FILE, so
  the tests failed on a missing import, not on the defective behavior. In
  truth the tests were non-discriminating: the buggy logic could be fully
  restored (call sites reverted, the fix branch deleted) with the suite
  staying green, because the fixtures' data happened to make wrong and
  right code produce identical output.
- **Why It Happened**: Every test hand-fed a correct input value instead
  of deriving it the way the real call site does, so the producer/consumer
  contract was never exercised; and the discrimination check removed the
  file instead of neutering the logic.
- **Solution**: Re-review caught it; replaced with a direct unit test of
  the helper plus component tests whose fixture data makes wrong and right
  outputs visibly differ; verified by surgically neutering ONLY the fix's
  logic in place (file, exports, imports intact) → exactly the new tests
  fail; restore → all green.
- **Preventive Rule**: Prove a regression test discriminates by reverting
  or neutering the specific defective logic IN PLACE and watching that
  test fail for the behavioral reason — never by deleting/stashing files
  (import errors prove nothing). When the bug is "call site passes the
  wrong value," the test must compute the value the way the call site
  does, or use fixtures where wrong and right visibly diverge.
- **Similar Situations**: Mutation-testing zero-kill tests; fixtures whose
  defaults coincide with the buggy path's output; snapshot tests
  regenerated from broken output; mocking the exact function whose
  behavior the test claims to cover (this session's other instance:
  monkeypatching the selection function made shuffle regressions
  undetectable by the HTTP tests).

### LL-0029 — Competing gesture listeners: `pointerdown` pre-empts the touch sensor whose delay was the whole mitigation

- **Root Cause**: Adding drag to controls that had to stay tappable, the
  sensor set was PointerSensor (activation distance 8) + TouchSensor
  (delay 150, tolerance 8), and every drag source got
  `touch-action: none`. On touch devices `pointerdown` fires *before*
  `touchstart`, so PointerSensor claimed every gesture and TouchSensor's
  delay/tolerance never applied — the intended "short swipe scrolls,
  long-press drags" behavior never existed at all. `touch-action: none`,
  which is what makes pointer-based touch dragging work in the first
  place, then blocked page panning anywhere a drag source sat. On a phone
  the control list filled most of the viewport, so scrolling over it
  simply stopped — a regression on 26 already-shipped items, introduced by
  a feature that never touched them beyond reusing their input component.
- **Why It Happened**: The plan named TouchSensor's `delay` as the
  scroll-vs-drag mitigation without checking event ordering, so a config
  option was trusted to arbitrate a race it never entered.
  `touch-action: none` was carried in from a drag recipe as "the thing
  that makes it work," with no one asking what it costs. Neither layer of
  testing could see it: jsdom has no compositor, and the automation
  browser reported `navigator.maxTouchPoints === 0`, so no synthetic touch
  test could reproduce panning either. It surfaced only when a
  whole-branch review read the CSS against the sensor choice.
- **Solution**: Replaced PointerSensor with dnd-kit's MouseSensor +
  TouchSensor pair — mouse keeps distance-8 so a stationary press-release
  stays a click, touch keeps delay-150/tolerance-8 so a short swipe
  scrolls and a long-press drags — and downgraded the sources to
  `touch-action: manipulation`. Verified by three proxies, with the real
  finding stated as unverified: computed `touch-action` on every drag
  source, a short synthetic swipe leaving every `touchmove`
  un-`preventDefault`ed and starting no drag, and a >150ms press
  correctly activating one. A real-device pan check was recorded as still
  outstanding rather than implied.
- **Preventive Rule**: When two listeners can claim the same gesture,
  choose them by event ordering, not by config intent — `pointerdown`
  precedes `touchstart`, so any PointerSensor pre-empts a TouchSensor's
  delay. Where touch scrolling must survive, use MouseSensor +
  TouchSensor. Treat every `touch-action: none` as an explicit decision to
  disable panning over that element's area, and say what it costs at the
  line you add it. When a suite provably cannot observe a behavior
  (`maxTouchPoints === 0`, no compositor), name the proxies you used and
  what remains unproven — never let green tests imply coverage the
  environment cannot produce.
- **Similar Situations**: `preventDefault()` on `touchstart` suppressing
  click synthesis; `user-select: none` / `overscroll-behavior` copied from
  the same recipes; passive-vs-active listener assumptions; retrofitting
  any gesture onto controls that already had a simpler interaction
  (dragging onto buttons, swipe onto a scroller); a resized desktop
  browser standing in for a real touch device; and generally any "the
  option handles it" belief about two handlers competing for one input.

### LL-0030 — Omitting a library's keyboard sensor removed the capability but not its promise to screen readers

- **Root Cause**: A drag implementation deliberately left out dnd-kit's
  KeyboardSensor, because its Enter/Space activation would have fought the
  buttons' existing click handlers. The library's *default*
  `screenReaderInstructions` still said "To pick up a draggable item,
  press the space bar," and the spread `{...attributes}` attached an
  `aria-describedby` pointing at that text to every draggable control. So
  assistive-tech users were instructed to press a key that no longer
  picked anything up — and space still activated the button, which
  **removed** a placed item or **cleared** a filled slot. The narration
  described a removed capability, and the key it named destroyed the
  user's work instead. `attributes` also stamped
  `aria-roledescription="sortable"`/`"draggable"` onto plain buttons that
  no longer were either.
- **Why It Happened**: Omitting a sensor is a local edit to behavior;
  the library's default ARIA narration is global and assumes the full
  sensor set, so removing the handler silently left the promise standing.
  Nobody asked what the library still *asserts* to assistive tech after
  its interaction model was deliberately narrowed. Coverage could not
  catch it either: the tests exercised tap paths and pure reducers, and
  none of them read `aria-describedby` — an entire output channel had no
  assertions anywhere.
- **Solution**: Passed explicit
  `accessibility={{ screenReaderInstructions: { draggable: … } }}` to
  every drag context, describing the interaction that actually exists
  (activate the control to place or remove; drag with mouse or
  long-press), and overrode `roleDescription` on the draggables. Doing so
  surfaced a second finding worth recording: the library's
  `aria-describedby` points at an id that does not exist in the rendered
  DOM, so neither the old misleading text nor the new correct text is
  actually announced. The original defect was therefore inert — by luck,
  not design — and the fix is currently unobservable for the same reason.
- **Preventive Rule**: When you deliberately omit part of a library's
  interaction model, audit what the library still *says* about it —
  default ARIA text, `roledescription`, help strings, keyboard hints —
  because deleting a handler does not retract a promise. Treat an
  announced-but-absent affordance as worse than a missing one: the key the
  instructions name almost always still does *something*, and that
  something is often destructive. And verify an accessibility fix by
  confirming the announcement reaches the accessibility tree, not merely
  that the string was passed to the right prop.
- **Similar Situations**: `aria-label` copied from a component that had
  more behavior; `role="button"` on a `div` with no key handler;
  focusable-but-disabled controls; tooltips and help text surviving the
  removal of the shortcut they describe; i18n strings outliving their
  feature; any library whose defaults narrate a config you have since
  narrowed (form validation messages, carousel/table a11y text); and more
  broadly, any output channel — ARIA, logs, telemetry, emails — that no
  test asserts on and therefore drifts from the behavior it reports.

### LL-0031 — A git worktree does not isolate the tooling's own config directory

- **Root Cause**: A task running inside a git worktree edited a file under
  the agent harness's config directory (`.claude/skills/**`). It committed
  correctly on the worktree's branch **and** left a byte-identical
  uncommitted copy in the *primary* checkout — so the default branch sat
  dirty with a change that belonged to a feature branch. This is not an
  incidental stray write: the harness resolves its config/skills directory
  from the primary working directory rather than the active worktree, which
  makes it systematic for that whole path class. Confirmed by reverting the
  primary copy and watching the skill metadata the harness reported switch
  back to the pre-change version in the same session.
- **Why It Happened**: Worktree isolation is genuinely real for tracked
  source, and that creates a false sense of total coverage — nothing in the
  setup signals that an entire category of path is shared with the primary
  tree. The evidence is also absent from where you would look for it:
  `git status` **inside** the worktree was clean, and nothing prompts you
  to inspect a tree you are not working in. Second occurrence of the
  wrong-tree class after LL-0026, which had flagged the symptom without
  identifying the mechanism.
- **Solution**: Proved the primary checkout's copy was byte-identical to
  the branch's committed version — `git diff <branch>:<path> -- <path>`,
  or equivalently `git hash-object <path>` against
  `git rev-parse <branch>:<path>` — which established the content was
  already safe on the pushed branch, then `git restore`d it so the change
  would reach the default branch through the pull request as designed.
- **Preventive Rule**: Treat a worktree as isolating tracked source only.
  Tooling and agent config directories — `.claude/`, editor state, hook and
  MCP config, anything the tool resolves from "the project root" — are
  effectively shared, so assume writes there land in the primary tree. Run
  `git status` in the PRIMARY checkout at the end of any branch whose work
  touched such a path. Before discarding a stray, *prove* it is
  byte-identical to the branch's committed version instead of assuming:
  identical means `git restore` loses nothing, and different means you are
  one command away from destroying unique work.
- **Similar Situations**: LL-0026's dev server launched from the wrong
  checkout (same worktree context, different mechanism); user-level or
  global tool config shadowing a repo-local one; a repo-scoped task
  mutating `~/.config`; `.venv` / `node_modules` symlinked or shared across
  worktrees; and the downstream cost if the stray goes unnoticed —
  committing it to the default branch conflicts with the very pull request
  that carries the identical change.

### LL-0032 — "Inconclusive under automation" was the defect, not a harness artifact

- **Root Cause**: During browser verification one interaction — dragging an
  already-placed item back to the source pool — would not complete; the
  drop kept resolving onto the element's own slot. It was recorded honestly
  as "inconclusive under automation," attributed to synthetic-event
  limitations, and waved through because a second interaction (tap to
  clear) covered the same user action. It was a real product defect: the
  drag library's default collision strategy compared distances between
  element *centers*, and with one large drop target competing against
  several small ones the nearest center was never the intended one. An A/B
  under an identical procedure settled it in a single pass — the default
  strategy dropped the item on the **wrong** target, while the
  pointer-position strategy resolved correctly.
- **Why It Happened**: "The harness cannot do this" is a comfortable
  explanation that terminates investigation, and it was propped up by two
  true facts: synthetic pointer events genuinely are lossy, and a working
  alternative path existed, which removed all pressure to explain the
  anomaly. The attribution to tooling was never itself treated as a claim
  requiring evidence. Worse, the anomaly's *shape* — a drop resolving back
  onto its origin — was equally consistent with both explanations, so
  nothing in the observation alone could separate them.
- **Solution**: A reviewer proposed the alternative collision strategy as
  an **experiment** rather than as a fix. Holding the procedure fixed and
  varying only the implementation discriminated immediately: default → item
  landed on the wrong target; pointer-based → correct. The strategy was
  changed for that context, the anomaly reclassified from "environment
  gap" to "defect, fixed," and the earlier verification note corrected.
- **Preventive Rule**: Treat "inconclusive because of the tooling" as a
  hypothesis, never a conclusion — it needs evidence like any other claim.
  When an observation is consistent with both a harness artifact and a
  product defect, discriminate by holding the procedure fixed and varying
  the implementation (A/B two configurations); differing outcomes convict
  the product. Be most suspicious precisely when a working alternative path
  lets you move on without explaining the anomaly — that convenience is
  what buys a defect its ticket to production. See rule 7 in
  `docs/standards/testing.md`.
- **Similar Situations**: "Flaky" used as a diagnosis rather than a symptom
  (rule 6 in the same standard); blaming CI for a failure that reproduces
  locally on the same inputs; dismissing a timeout as network noise;
  attributing a visual difference to the screenshot pipeline; any "known
  limitation" inherited without a reproduction; and LL-0028's
  non-discriminating regression tests — the same family, where the evidence
  on hand cannot distinguish two live hypotheses.

### LL-0033 — Per-task review cannot see an omission, because no task's diff contains it

- **Root Cause**: A 13-task feature added a new item type to an app that
  renders a human-readable badge for each type from a `TYPE_LABELS` map. No
  task's brief mentioned that map, so no task touched it, so it appeared in
  no task diff — and all thirteen task-scoped reviews passed. The new type
  fell through to the raw enum string, rendering a lowercase `highlight`
  badge beside properly title-cased siblings. Only the whole-branch review,
  reading the feature as a unit rather than as a sequence of diffs, caught
  it.
- **Why It Happened**: A task-scoped review is defined by a diff, and a diff
  is the wrong instrument for a *missing* edit. Reviewers were asked "does
  this change do what its task says, and is it well built?" — both answered
  correctly, and neither could answer "is there somewhere else this type
  should have been registered?" The plan inherited the same blind spot: it
  was written by enumerating the files the feature adds, which is exactly
  the enumeration that omits a file nobody thought of. Registry-style code
  (label maps, dispatch tables, type unions, factory switches) is where this
  concentrates, because adding a variant obliges edits in places the
  variant's own implementation never references.
- **Solution**: The whole-branch review, handed the branch as one package
  and asked explicitly about cross-task integration rather than per-task
  correctness, found it in a single pass. Fixed in the final fix wave along
  with the other accumulated findings rather than as its own dispatch.
- **Preventive Rule**: Never let per-task reviews stand in for a broad pass
  — they are structurally incapable of catching an omission, however many of
  them pass. When adding a variant to an existing set (a new enum member,
  item type, message kind, provider), grep for an existing sibling *by name*
  across the whole tree before writing the plan, and treat every hit as a
  candidate task; the sibling's name finds the registries the new variant's
  own design never mentions. Give the final review the whole branch and ask
  it specifically for cross-task integration and consistency with siblings,
  not for a re-audit of what the task reviews already covered.
- **Similar Situations**: A new enum variant missing from a `switch` with a
  permissive `default`; a new locale absent from a language picker; a new
  event type never registered with its dispatcher; a new model class not
  imported into migration autogenerate metadata; a new feature flag missing
  from the admin listing; DI containers, plugin registries, CLI subcommand
  tables, serializer maps — and generally any change whose correctness
  depends on a file the change itself gives no reason to open.

### LL-0034 — An implementation plan's claims about the codebase are assertions, and four of thirteen were false

- **Root Cause**: A plan written to be executed by fresh subagents embedded
  factual claims about the existing tree — which files exist, what imports
  what, what a fixture already provides. Four of its thirteen task briefs
  were wrong: a Pydantic fixture omitted seven fields the model requires; a
  claim that "nothing outside this module imports these three helpers" was
  false; two test files the briefs told implementers to append to did not
  exist at all; and one step mandated an edit that would have made the same
  task's other half dead code. Every defect was caught by the implementer or
  its task reviewer, so nothing shipped broken — but each cost a dispatch
  cycle.
- **Why It Happened**: The claims were produced while reading the codebase
  and then never re-verified, so authoring confidence carried into the plan
  as though it were evidence. The import claim is the instructive one: it
  came from grepping two import-path spellings when a third was in use, and
  a *negative* result from an incomplete search is indistinguishable from a
  true absence. Negative claims are the ones a plan states most confidently
  and verifies least, because nothing in the authoring flow forces a second
  look. Compounding it, a subagent receives only its own brief, so a false
  claim is authoritative to its reader — there is no surrounding context to
  contradict it.
- **Solution**: Implementers reported the discrepancies instead of coding
  around them, and the controller verified each against the tree before
  accepting the deviation. The false import claim was settled by grepping
  the *symbol* rather than any path spelling; the missing test files by
  listing the directory.
- **Preventive Rule**: Treat every factual claim in a plan as a claim
  requiring evidence, and verify them in one pass *before* the first
  dispatch rather than lazily as each task reaches them. Verify negatives by
  searching for the symbol or filename itself, never for one spelling of a
  path that would reference it — "nothing imports X" and "file Y does not
  exist" are the two highest-risk sentences a plan can contain. Where a
  brief asserts that a fixture or helper already exists, name it and say
  where, so the implementer fails fast instead of building on a phantom.
  And tell implementers explicitly that the brief may be wrong and that
  reporting a contradiction beats satisfying it.
- **Similar Situations**: "This function has no other callers" before a
  signature change; "this column is unused" before a migration; "no test
  covers this" before a rewrite; a runbook citing a path that has moved; any
  dead-code deletion justified by a single grep; and LL-0028's family more
  broadly — evidence that cannot distinguish "absent" from "not looked for
  properly."

### LL-0035 — Dropping a superseded table armed a latent 500 in the fallback path that still read it

- **Root Cause**: A legacy single-blob `tasks` table had been fully migrated
  into a per-item `items` table and was being kept only as a rollback. The
  migration helper that read it, `migrateLegacyBlob`, still ran on every
  `GET /api/tasks/sync` where the user had **zero** items, and the shared
  PostgREST helper throws on any non-OK response. Dropping the table would
  therefore have converted "empty tracker" from an empty list into a hard 500
  — not at drop time, but the first time anyone reached that branch.
- **Why It Happened**: The table looked inert. Reconciliation proved every row
  had migrated, which answers "is the data safe to delete?" but not "does
  anything still read this?" — two different questions that feel like one. The
  reader was also invisible to the obvious check: grepping for the table name
  returned dozens of hits because `tasks` is the domain noun, so the single
  real reference was buried in noise from variables, routes, and prose. And
  the path is cold by construction — it only fires for a user with no rows, so
  no amount of ordinary use would have surfaced it.
- **Solution**: Found the reader before dropping, and shipped a guard *first*
  that treats a missing relation as "nothing to migrate" — matching PostgREST's
  `PGRST205` and Postgres' `42P01` specifically, not a bare 404 and not a
  blanket catch. Two tests: one that a dropped table migrates nothing, one that
  a 503 still propagates. Only then was the table dropped, and the external
  check re-run to confirm the anon key still returned `[]`.
- **Preventive Rule**: Before dropping or renaming a table, enumerate its
  *readers*, not just its rows, and do it by searching for the access
  expression — the query string, ORM call, or `from("x")` — rather than the
  bare table name, which collides with the domain vocabulary. Pay particular
  attention to fallback, migration, and first-run paths: they are cold, rarely
  tested, and disproportionately likely to reference exactly the legacy thing
  being removed. When guarding such a path, match the specific error code for
  "this object no longer exists"; a catch-all disguises an outage as the benign
  case, which is strictly worse than the crash it replaces.
- **Similar Situations**: Deleting a feature flag whose default branch is the
  cold one; removing a config key still read by an installer; dropping a column
  a backfill job selects; retiring an endpoint a mobile client calls only on
  first launch; deleting a seed file referenced only by a fresh-database path.
  Related to LL-0034's negative claims — "nothing reads this" is exactly the
  sentence stated most confidently and verified least.

### LL-0036 — A backup that cannot be dumped mechanically must be proven against its source, not eyeballed

- **Root Cause**: A rollback copy of a table was needed before an irreversible
  drop, but there was no direct dump path available — no local credentials for
  the database, and the query tool returned results into a transcript rather
  than to a file. The backup therefore had to be reconstructed by hand from
  query output, which is exactly the situation where a silently dropped record,
  a mangled escape sequence, or a truncated long field produces a file that
  looks complete and is not.
- **Why It Happened**: Backups are treated as a checkbox before destructive
  work — the act of producing one feels like the safety, so its *fidelity*
  goes unexamined. The risk concentrates in the least readable fields
  (embedded newlines, quoted URLs, unicode) which are the ones a human scan
  skips, and record count matching gives false comfort because it is the one
  property that survives most corruption.
- **Solution**: Computed a per-record fingerprint on both sides over every
  field — including hashes of the long free-text fields and an ordered digest
  of nested arrays — then compared aggregate md5s. They matched, which proved
  the transcription faithful before anything was dropped. The definition was
  written to be reproducible in both SQL and the target language rather than
  relying on either side's default serialization, since JSON key ordering and
  whitespace differ between them.
- **Preventive Rule**: Never let "a backup exists" stand in for "the backup is
  correct" when the copy was produced by any means other than a mechanical
  dump. Verify it against the live source with a content fingerprint that
  covers every field, and define that fingerprint yourself from primitives —
  do not compare serialized forms across two systems, because formatting
  differences produce mismatches that look like corruption and send you
  hunting a phantom. Record counts and spot checks are not verification.
  Verify *before* the destructive step, while the source still exists to
  compare against.
- **Similar Situations**: Copying credentials or config between environments
  by hand; reconstructing a fixture from log output; transcribing values from
  a screenshot or PDF; any CSV round-trip through a spreadsheet, where type
  coercion silently rewrites long numeric ids and dates; migrating data via
  copy-paste because the export button is missing.

### LL-0037 — Mocking a client wrapper at every call site leaves the store's own configuration untested

- **Root Cause**: A vector collection was created without payload indexes on
  the two fields the application filters by. The engine rejects a filter on an
  unindexed field outright (HTTP 400, "Index required but not found") rather
  than degrading to a scan, so *every* filtered call had been failing since the
  feature shipped. The unfiltered operations — write and similarity-search —
  worked perfectly, which is why the feature demoed, shipped, and ran for days
  looking healthy. The broken call was the delete-by-filter used to retract
  content, i.e. the one path that only runs when a human has just decided the
  content is wrong.
- **Why It Happened**: Every test mocked the thin client wrapper's functions
  wholesale, which is the sane way to test callers — but it means no test ever
  produced a real request, so no test could observe that the store's schema
  didn't support the query. The configuration was in nobody's diff: the
  collection was created once, imperatively, by a setup call whose arguments
  said nothing about indexes. The one operational script that touched the live
  store happened to use only the two unfiltered operations. Catching this
  needed a call that was both *real* and *filtered*, and nothing in the suite
  or the tooling was both.
- **Solution**: Moved index creation into the same `ensure_collection` step
  every code path already calls, so any execution repairs the configuration.
  Added a test that ties the set of indexed fields to a module-level constant
  naming the fields the filters actually use — so adding a filter on a new
  field fails in CI rather than in production.
- **Preventive Rule**: When a thin client wrapper is mocked at every call site,
  the external store's configuration is untested *by construction* — assert the
  configuration itself. Make the idempotent setup step create everything the
  queries need (indexes, constraints, permissions), and pin "what we filter on"
  to "what we index" in a test. A mock will never tell you the two disagree.
  Corollary: treat "reads fine, writes fine" as covering a strict subset of the
  API — the operations that only fire on rare paths (retraction, deletion,
  rollback) are exactly the ones no happy path exercises.
- **Similar Situations**: A missing database index that only breaks a rarely-hit
  query plan; an object-storage bucket lacking a lifecycle or CORS rule used by
  one endpoint; a search index missing a facet field; a message queue without
  the dead-letter binding the error path publishes to; any IAM permission needed
  only by a cleanup job. Also: any store whose setup is imperative and one-time
  rather than declarative and re-applied.

### LL-0038 — A finding that confirms the bug you just diagnosed deserves more scrutiny, not less

- **Root Cause**: Two services were configured from independent environment
  variables that, in a normal development shell, point at *different
  environments*: the database URL at the local stack, the vector store URL at
  the cloud instance (no local instance of it exists). An operational script
  read content from one and wrote to the other, so it took **local** state as
  truth and applied it to **production**. The specific column it filtered on —
  review/publication status — is precisely the column known to drift between
  local and cloud, because reviews happen against the deployed app and revert
  locally on any database reset.
- **Why It Happened**: This was found immediately after diagnosing a real bug
  whose predicted symptom was "unreviewed content sitting in the production
  index." Local then reported exactly that. The finding *fit the freshly-built
  model perfectly*, so it read as confirmation rather than as a claim needing
  its own check — and the content was deleted from production. Cloud had it
  published; it was legitimate. The single question that would have caught it,
  "which database am I actually reading?", went unasked **because** the story
  already hung together. Confirmation bias is at its strongest right after a
  correct diagnosis, when a new fact slots neatly into a model that just proved
  itself.
- **Solution**: Restored the deleted content from the authoritative (cloud)
  row. Verified the blast radius rather than assuming it: hashed the same field
  across both databases for every affected row and confirmed they matched
  everywhere except the one genuinely-drifted record. Added a guard the
  operational scripts call before reading anything — it prints both endpoints
  and exits on a local-source/production-target mismatch unless an explicit
  `--allow-local-source` flag is passed. Corrected the already-pushed commit
  message and lessons entry additively rather than rewriting history.
- **Preventive Rule**: Before a script that mutates production reads anything,
  make it state which environment *each* endpoint points at, and refuse a
  mismatch by default. Two independently-configured services are two
  independent chances to be in the wrong environment, and the failure is silent
  because both connections succeed. Separately, and more generally: when a new
  finding confirms the theory you just formed, that is the moment to verify its
  provenance, not to act on it. Treat "this is exactly what I predicted" as a
  prompt to ask what else could produce the same observation.
- **Similar Situations**: Any script pairing a database read with a cache,
  search index, queue, or object-store write; running a migration against
  staging data but production schema; a feature-flag service pointed at one
  environment while the app points at another; seeding a production system from
  a local fixture; any destructive cleanup whose target list is computed from a
  different source of truth than the one being cleaned.

---

### LL-0039 — Deriving calendar dates via toISOString() is wrong east of UTC, and UTC-only tests can't see it

- **Root Cause**: Date helpers (`todayStr`, `addDays`) in Task Command built a
  Date at *local* midnight (`new Date("YYYY-MM-DDT00:00:00")`) and then
  formatted it with `toISOString().split("T")[0]`, which formats in *UTC*. East
  of Greenwich, local midnight is still the previous day in UTC, so in Naples
  (UTC+2) `addDays(d, 1)` returned `d` and `addDays(d, 0)` returned the day
  before. Every snooze, follow-up default, and recurrence was silently one day
  early. Two schedulers compounded it: Vercel functions run in UTC and the n8n
  container ran America/New_York, so the "0600 brief" fired at noon local.
- **Why It Happened**: The mixed idiom (parse local, format UTC) looks
  symmetrical and passes review. The dev machine and every test ran in UTC,
  where local and UTC dates coincide and the bug is mathematically invisible.
  The user's actual timezone was recorded in project memory but was never
  treated as an input to code correctness — only as a scheduling fact.
- **Solution**: Format calendar dates from local date fields
  (`getFullYear`/`getMonth`/`getDate`), never `toISOString()`. Pin every
  server-side "what day is it for the user" computation to an explicit IANA
  zone held in one config point per host (env var on Vercel,
  `settings.timezone` + a mirrored constant in the n8n Code node). Do pure
  calendar *arithmetic* on `Date.UTC(...) + n * 86400000`, which is DST-free —
  adding a day of milliseconds to a wall-clock instant loses an hour across
  spring-forward and lands a day late.
- **Preventive Rule**: Any code that produces a calendar date (`YYYY-MM-DD`)
  must have tests that run it under multiple `TZ` values — at minimum UTC, one
  zone east (`Europe/Rome` or `Pacific/Auckland`), one west
  (`America/Los_Angeles`), and both DST switch days. A date-handling suite that
  only runs in UTC proves nothing. Grep candidates: any
  `toISOString().split('T')` or `toLocaleDateString()` without an explicit
  `timeZone` option.
- **Similar Situations**: Cron schedules in hosted runners (n8n, GitHub
  Actions, Vercel cron) — the cron's firing zone and the job's own date math
  are configured in *different places* and must be changed together; "due
  today"/"overdue" predicates in any tracker; daily-rollup emails; date-range
  filters passed to APIs; anything comparing a stored `YYYY-MM-DD` against
  "now".

### LL-0040 — Auth middleware matchers exclude by file extension, so extensionless well-known routes get gated and break silently

- **Root Cause**: A Clerk/Next middleware matcher excluded static assets by
  extension (`...\.(?:png|svg|ico|css|js|woff2?)...`) and protected everything
  else. Framework-generated well-known routes have no extension —
  `/manifest.webmanifest` matched no exclusion, and a generated OG image
  serves from `/opengraph-image?<hash>` — so both were treated as app pages
  and redirected to sign-in.
- **Why It Happened**: The consumers of those routes are never signed in and
  never complain: an OS install prompt and a link-preview crawler. Every human
  test passes, because a developer testing the site is authenticated — the
  feature is broken *only* for the audience it exists for. Extension-based
  exclusion silently encodes "assets have extensions", which stopped being
  true once frameworks began generating manifests and social images as routes.
- **Solution**: Add the extensionless public routes to the middleware's public
  list explicitly (`/manifest.webmanifest`, `/opengraph-image(.*)`), and verify
  unauthenticated with
  `curl -o /dev/null -w "%{http_code} %{content_type}"` — a browser check by a
  logged-in developer proves nothing here.
- **Preventive Rule**: After adding any framework-generated metadata route
  (manifest, `opengraph-image`, `robots.txt`, `sitemap.xml`, `.well-known/*`,
  RSS), curl it **with no cookies** and assert both status and content type.
  If the app has auth middleware, assume the route is gated until proven
  otherwise.
- **Similar Situations**: Health-check and webhook endpoints behind auth
  middleware; `.well-known/apple-app-site-association` and `assetlinks.json`
  (deep links fail with no error surface); Stripe/GitHub webhook receivers;
  anything an unauthenticated *machine* fetches.

### LL-0041 — A gated affordance must state its gate where the user already looks, not in a hover tooltip

- **Root Cause**: A milestone chip ("80% accuracy") stayed muted for a user
  holding 85% accuracy, because the rule also required a 50-graded-answer
  floor he had not reached. That requirement existed only in a native `title`
  tooltip.
- **Why It Happened**: The tooltip felt like sufficient disclosure at
  authoring time. It isn't: `title` needs a long motionless hover on desktop
  and never fires on touch, so in practice the rule was invisible. A
  correctly-gated element whose gate cannot be seen is indistinguishable from
  a broken one — and the user's first hypothesis is "bug", not "unmet
  requirement", which burns their trust and your debugging time.
- **Solution**: Render progress inline on the element itself ("13/50 graded"),
  and when several conditions gate it, show *whichever one is actually
  blocking* — telling a user at 85% that they need 80% accuracy is worse than
  saying nothing. Keep the tooltip as a bonus channel, never the only one.
- **Preventive Rule**: Any disabled/locked/muted UI state must make its unlock
  condition visible without hover, focus, or a click. If the state is computed
  from more than one threshold, the visible text names the binding one.
- **Similar Situations**: Disabled submit buttons ("why can't I continue?");
  greyed-out menu items; rate-limited actions; feature flags and plan gates;
  form fields whose validation rule only appears on error; anything with a
  `title` attribute doing load-bearing explanatory work.

### LL-0042 — A round-trip through your own decoder cannot validate an encoder whose real consumer is someone else's parser

- **Root Cause**: Task Command RFC 2231-encoded MIME attachment filenames with
  `encodeURIComponent`, which leaves `'`, `(`, `)` and `*` unescaped. `'` is the
  delimiter in `charset'language'value`, so `città's report.pdf` ended its own
  `filename*=` parameter early and would arrive truncated in the recipient's
  client. Two assertions looked like coverage and neither could fail: one
  checked only that the output contained the prefix `filename*=UTF-8''`, and the
  other asserted `decodeURIComponent(value) === original`, which passes on the
  broken output too.
- **Why It Happened**: Both assertions tested the producer against itself. A
  round-trip pairs an encoder with its own matching decoder, so any character
  the pair silently agrees to leave alone is invisible to it — the check is
  symmetric in precisely the way the bug is. `decodeURIComponent` is lenient
  about characters RFC 5987 forbids; the actual consumer, a mail client's RFC
  2231 parser, is not. The prefix check compounds it by confirming the *shape*
  of the output while asserting nothing about its content.
- **Solution**: Assert the full encoded value against a literal expected string
  (`citt%C3%A0%27s%20report.pdf`); keep the round-trip only as a secondary
  check; add a negative assertion that the delimiter cannot appear raw in the
  value; and assert that characters the spec *does* permit (`!` and `~` are
  attr-char) are still left alone, so the fix isn't over-escaping. Discriminated
  per LL-0028 by neutering only the escape in place — strict equality failed and
  the round-trip stayed green, which is the whole lesson in a single run.
- **Preventive Rule**: When testing an encoder whose consumer is an external
  parser — MIME headers, URLs, CSV, shell quoting, SQL identifiers, JSON
  embedded in HTML — assert the literal encoded output against the spec's
  grammar. Never let a round-trip through your own decoder be the primary
  assertion: it proves a closed loop, not conformance. Add a negative assertion
  naming the delimiter or metacharacter that must never survive raw, and treat
  `.includes(prefix)` on encoder output as no coverage at all.
- **Similar Situations**: Shell argument quoting verified by re-splitting with
  the same splitter; CSV escaping read back with the same library; SQL
  identifier quoting; HTML/JS escaping checked with a lenient parser; base64url
  vs standard base64 where one alphabet is invalid downstream; any
  `encodeURIComponent` used for something that is not a URI component.

### LL-0043 — Config that reads correctly in every dashboard can still be broken; only the live end-to-end flow proves it

- **Root Cause**: Cutting an auth provider over to a production domain (custom
  domain + live keys) looked complete after checking every config screen — DNS
  records verified, TLS issued, env vars updated and redeployed. Two real
  breaks were invisible from that review. (1) Social sign-in failed with
  "Missing required parameter: client_id" because the provider's production
  tier requires the developer's own OAuth client, while its dev tier silently
  uses the vendor's shared test credentials — nothing in the dashboard flags
  this until sign-in is actually attempted. (2) The backend's CORS allowlist
  was never actually updated to the new frontend origin (an edit made earlier
  in the session had been lost/skipped), so every post-sign-in API call failed
  with a generic browser "Failed to fetch" — no error surfaced in either
  system's own logs, because CORS rejection happens silently at the
  browser/network layer.
- **Why It Happened**: Both gaps live at the *seam* between two independently
  configured systems (IdP↔OAuth provider, frontend origin↔backend CORS), where
  each side's own dashboard shows a state that is locally valid but proves
  nothing about whether the seam actually works. Config review is a checklist
  over what you can see; an integration only reveals itself by running.
- **Solution**: After any multi-system cutover (auth provider, domain
  migration, API/CORS boundary), drive the real user path in a browser — sign
  in, load a page that requires the new integration, check the network tab and
  console for errors — before declaring it done. Config screens are a
  necessary check, not a sufficient one.
- **Preventive Rule**: A cutover that changes how two independently configured
  systems talk to each other (OAuth provider ↔ IdP, frontend origin ↔ backend
  CORS, webhook URL ↔ receiver, DNS ↔ TLS cert) is not verified until the
  actual cross-system call has been exercised live, not just inspected in each
  system's own dashboard.
- **Similar Situations**: Any OAuth/SSO provider cutover between dev and prod
  tiers; CORS/allowed-origins lists after a domain change; webhook endpoint
  URLs after a redeploy; DNS cutovers where "the record looks right" is
  checked before "the cert actually issued and the handshake completes"; any
  config split across two admin consoles with no single source of truth.

---

### LL-0044 — A review-state backfill must replay every field the real review flow sets, not just the status column

- **Root Cause**: A `seed.sql` backfill block persisted a batch of
  AI-generated content's `clinical_review → published` transition (so a
  local `db reset` wouldn't revert review state that only exists on the
  deployed/cloud database) by copying the `current_stage` column change
  directly. It did not also replay the side effect the real review UI
  performs when publishing: flipping each evidence citation's `verified`
  flag from `false` to `true`. A DB-level publish-readiness trigger required
  at least one verified reference before `current_stage` could become
  `published`, so the backfill's own `UPDATE` failed its own gate — breaking
  `supabase db reset` for the whole project from that commit onward, silently,
  because nobody happened to run a full reset again until a later session.
- **Why It Happened**: The backfill was written by reasoning about "what
  changed" (a status field) rather than "everything the real action does."
  A status transition driven by a UI action is rarely just one column;
  replaying only the visible field misses the invariants the real action
  also satisfies, invariants a DB trigger can then enforce and reject.
- **Solution**: Query the source of truth (the deployed/cloud database,
  where the real action executed) for the *actual* resulting row shape,
  not just the field being backfilled, then compose the backfill's SQL from
  that. Verify the backfill by running the exact reset/replay procedure end
  to end (`db reset`, not a row-count spot check) before trusting it.
- **Preventive Rule**: Any script or migration that persists a state
  transition normally driven by application logic (a review approval, a
  publish action, a status flip enforced by a DB trigger/constraint) must
  replay every side effect that transition performs, not only the field
  being backfilled — and must be verified by actually re-running the
  process that would expose a missing side effect (a full environment
  reset/rebuild), not by checking the row count or the one field you
  intended to change.
- **Similar Situations**: Any "persist reviewed/approved state so a reset
  doesn't lose it" backfill in a system with DB-level readiness gates,
  soft-delete/audit trail side effects, or multi-column state machines;
  any migration that mimics an application-level action from outside the
  application.

### LL-0045 — An unpaginated `.select()` against a growing table silently truncates once it crosses the API's default row cap

- **Root Cause**: A production code path (and its test-suite equivalent)
  fetched "all rows matching a filter" from a Postgres table via
  PostgREST/Supabase's Python client with a single `.select().execute()`
  call and no `.range()`/pagination. PostgREST defaults to capping
  unbounded selects at 1000 rows. The table in question started well under
  that limit, so the code worked correctly for months; once a content-growth
  operation pushed the filtered row count past 1000, the call started
  silently returning only the first 1000 rows with no error — for
  production code, that meant a personalization feature quietly lost access
  to the newest ~5% of its data pool for every user who exercised that code
  path.
- **Why It Happened**: The code was correct when written and stayed
  correct for a long time, so nothing about it looked wrong in review; the
  bug is purely a function of data volume crossing an implicit, undocumented
  (from the call site's perspective) threshold, which nothing in the code
  or its tests encoded as an assumption to watch. The test suite's own
  helper had the identical unpaginated-select bug, so it silently
  undercounted the expected total right alongside the real bug and never
  caught it — two independent instances of the same shortcut cancelling
  each other's symptom out. It also went unnoticed for a stretch because
  the local dev-environment reset was independently broken (see LL-0044),
  so nobody re-ran the test suite against fresh data in between.
- **Solution**: Paginated the fetch with `.range(start, start + PAGE - 1)`
  in a loop until a page returns fewer than `PAGE` rows, in both the
  production function and the test helper that was masking it.
- **Preventive Rule**: Any `.select()` (via PostgREST/Supabase or any other
  API with a default response-size cap) that is meant to return "every row
  matching a filter" — not a bounded/paged UI query — must paginate
  explicitly, even when today's row count is comfortably under the cap.
  When a table's row count crosses a round-number threshold (1000, 10000)
  as part of a content-growth or migration task, treat that as a trigger to
  grep the codebase for unpaginated selects against that table before
  declaring the task done, not just to run the existing test suite (which
  can have the same blind spot, as it did here).
- **Similar Situations**: Any "fetch all X" helper against a table expected
  to grow — user lists, content banks, audit logs, notification queues —
  built against any API with an implicit page-size default (PostgREST,
  most REST APIs with a default `limit`, GraphQL connections with a default
  `first`). Test helpers that mirror production query shapes are exactly as
  likely to carry the same latent bug as the production code.

### LL-0046 — An AI provider's default terms often claim a training licence on customer content, and the opt-out is gated behind a paid account

- **Root Cause**: A published privacy policy stated that user content was
  not used to train AI models and that no provider was permitted to do so
  on the operator's behalf. The claim was written from the operator's own
  intent and from familiarity with one provider's terms, not from reading
  each provider's terms. One provider in the chain — an embedding API that
  received the user's raw typed question on every request — granted itself,
  by default, "a worldwide, irrevocable, perpetual, royalty-free,
  fully paid-up right and licence to … train, improve, and otherwise
  further develop the Service" on customer content. Opting out was
  possible, but required a payment method on file, and the project was on
  the provider's free tier. The published claim was therefore false from
  the day it went live.
- **Why It Happened**: The best-known provider in the stack (Anthropic)
  bars training on API inputs in its commercial terms, which makes
  "providers don't train on API data" feel like an industry norm rather
  than a per-vendor fact. Embedding and other "utility" providers attract
  far less scrutiny than the model that visibly generates output, even
  though they receive the same raw user text. The free tier compounds it:
  the cheapest plan is often the one with the broadest data-use grant, and
  the opt-out is a paid-account feature, so the projects least able to
  afford review are the most exposed.
- **Solution**: Read each provider's current terms before publishing any
  data-use claim; traced what is actually transmitted to each provider at
  the call site rather than trusting the architecture diagram; corrected
  the policy to describe only the operator's own conduct until the opt-out
  was exercised; and made the corrected claim name each provider and the
  specific mechanism, so it is checkable rather than atmospheric.
- **Preventive Rule**: A privacy or marketing claim about what third
  parties do with user data is a claim about their contracts, not about
  your intentions, and must be sourced to their current terms with the
  effective date recorded. Before publishing "we don't train on your data",
  enumerate every service that receives user-derived content — including
  embedding, search, logging, analytics, error-reporting, and support-desk
  vendors — and check each one's default data-use grant and whether its
  opt-out has a plan or payment prerequisite. Where a claim cannot be
  sourced, narrow it to your own conduct ("we do not authorise…") or
  disclose the exception; never let the absolute version ship because the
  narrow version reads weaker. Re-check on any provider or plan change,
  and treat "free tier" as a signal to read the data-use clause first.
- **Similar Situations**: Any user-facing claim that depends on a third
  party's behaviour rather than your own code — subprocessor lists,
  "we never sell your data", data-residency and regional-processing
  statements, retention periods that a vendor actually controls, security
  certifications inherited from a host, and uptime or deletion guarantees
  passed through from an upstream service. The same plan-gating trap
  appears wherever a compliance control is a paid feature: DPAs that bind
  only on Pro/Enterprise tiers, audit logs, SSO, zero-retention modes, and
  regional pinning. Checking one vendor and generalising to the rest is the
  failure mode in every case.

### LL-0047 — An agent told to copy data verbatim can paraphrase it instead, and every structural check will pass

- **Root Cause**: A purely mechanical data-movement step — execute ~20
  pre-generated SQL `INSERT` statements verbatim against a database — was
  delegated to an LLM subagent whose prompt explicitly said to paste each
  statement exactly and not modify it. The agent altered one word of the
  payload anyway: a content string that read "noisy room" in the source file
  arrived in the database as "loud room". A semantically equivalent, "nicer"
  synonym, in one field of one row out of hundreds. The agent reported success
  and its own self-verification passed.
- **Why It Happened**: Every guard the pipeline had was a *structural* guard,
  and this defect is not structural. Row counts, per-group distribution,
  schema/key-parity audits, orphan-reference checks, database constraint
  triggers and JSON validity all passed, because the paraphrase is still
  grammatical English, still valid SQL, and still satisfies every constraint.
  The unstated assumption underneath the whole method was that **a bad copy
  fails loudly** — a mangled statement won't parse, which localizes the error
  for free. That holds for corruption and truncation. It does not hold for an
  LLM, whose characteristic failure mode is not mangling the text but
  *improving* it. A semantically valid paraphrase is invisible to every check
  that isn't comparing bytes, including a human reading the result, who sees
  a sentence that reads perfectly well.
- **Solution**: Detected by hashing **each content column separately** on both
  sides and comparing, rather than one whole-row or whole-table hash:

  ```sql
  select 'col_a', md5(string_agg(id||chr(1)||col_a::text, chr(2) order by id)) from t
  union all select 'col_b', md5(string_agg(id||chr(1)||col_b::text, chr(2) order by id)) from t
  -- ...one row per content column
  ```

  Per-column is what makes it debuggable: six of seven columns matched
  byte-for-byte immediately, isolating the defect to one column, after which
  bisecting (group → numeric bucket → row) localized the exact row in about
  six queries total, without ever pulling the content itself into context.
- **Preventive Rule**: When a step is *mechanical* — move these bytes from A to
  B unchanged — do not make the agent the transport. Have the agent write and
  run a script that moves the bytes, so the content passes through code rather
  than through a model's context. In the same session, an agent that streamed
  rows to a file via a script it wrote introduced zero drift, while the agent
  acting as the transport itself corrupted a word. Where an agent must be the
  transport, the completion check is a byte-level comparison (per-column
  hashes, `diff`, checksums) run after every load — a row count, a structural
  audit, and the agent's own self-report are all compatible with corrupted
  content and prove nothing about fidelity.

  Generalized: **the checks that pass are rarely the checks that matter.** Ask
  what layer the data has to survive — bytes, schema, serialization, the
  consumer — and verify at *that* layer. The corollaries below are two failures
  of the same shape, one caught by hashing bytes, one that should have been
  caught by instantiating the API's own model.
- **Similar Situations**: Any agent-mediated copy where the payload is
  human-readable prose and therefore paraphrasable — seeding or migrating
  content between environments, transcribing config or credentials between
  files, "just reformat this" edits, translating a fixture into another
  format, filling a template from a source document, or relaying a quoted
  message. The risk rises exactly where the content reads like natural
  language and falls where it is opaque (hashes, ids, base64), because there
  is nothing for the model to want to improve.

  **Corollary — when two copies disagree, don't assume which is authoritative.**
  Look for an edit or audit record to decide the direction of the fix. In this
  incident the same diff contained one divergence caused by agent corruption
  (source authoritative, correct the destination) and a second, opposite one:
  a genuine human edit made in the application UI that had never been
  persisted back to the source file, so every environment reset silently
  reverted it (destination authoritative, back-port to source). An audit
  trail — a `*_versions` table, `updated_at`/actor columns, application logs —
  distinguishes the two; guessing a direction repairs one row and destroys
  another.

  **Corollary — validity at the storage layer is not validity at the serving
  layer, and a list endpoint fails atomically.** Same session, same batch, a
  second "every check passed" defect with a different root cause. The agents
  drafting the batch wrote `"year": ""` for citations whose year they could not
  source. The column was untyped (`jsonb`), so the database stored it without
  complaint and *every* guard passed again — schema validation, key-parity
  audits, reference-integrity checks, and two separate constraint triggers. But
  the API's response model declared that field `int | None`, and the serializer
  could not coerce `""` to an integer. Nothing failed at write time; the defect
  surfaced only when a row was first *served* — and because the endpoint
  serializes a list, **16 bad values across 12 rows returned a 500 for all 200
  rows**, denying the human reviewer access to the entire queue.

  Two rules fall out. First: when data must survive a round trip through an API,
  validate it against **the model that will serialize it**, not only against the
  table it is written to — instantiate the response type over the loaded rows as
  the completion check. Untyped columns (`jsonb`, `json`, text blobs, schemaless
  documents) are precisely where the storage contract and the API contract
  silently diverge, because the database has agreed to enforce nothing. Second:
  weigh blast radius when a defect lands in a collection endpoint — one
  malformed row does not degrade one item, it takes down every item served
  alongside it, so "only 12 of 200 rows are affected" badly understates the
  impact. Read the production error trace before theorizing; it named the exact
  field path and route in one line, after size and pagination hypotheses had
  already been raised and would have wasted the investigation.

### LL-0048 — Over-escaping a quote inside a nested literal corrupts the stored text, and only a byte comparison sees it

- **Root Cause**: Patching a `jsonb` column through a database API by sending a
  JSON object literal nested inside a single-quoted SQL string. An apostrophe
  in the *content* needs exactly one level of SQL escaping (`''`); the escaping
  was applied twice (`''''`), once for the SQL literal and once again "for the
  JSON". JSON has no apostrophe escaping at all, so the second doubling was
  not neutralised by anything — it became literal characters. `"Broca''''s"`
  in the statement stored the text `Broca''s` in the database. The identical
  sentence written directly into a `.sql` seed file was correct, because there
  it sat in only one escaping context, so the file and the remote database
  silently disagreed on the content of two fields.
- **Why It Happened**: The same content was hand-authored twice, in two
  different quoting contexts, and only one of those contexts escapes
  apostrophes. Nested-delimiter bugs normally announce themselves by failing to
  parse, which trains the reflex that "extra escaping is the safe direction."
  That reflex is exactly wrong here: over-escaping produces output that is
  valid at *every* layer — valid SQL, valid JSON, valid UTF-8, grammatical
  English — and so passes silently. Every downstream guard was a structural
  guard and all of them passed: the JSON parsed, cross-references between JSON
  objects resolved, a referential-integrity helper was satisfied, the row count
  was right, and the row validated cleanly through its Pydantic response model.
  Model validation in particular is seductive and proves nothing here — it
  checks shape, not characters.
- **Solution**: Caught by hashing each content column separately on both sides
  and comparing (the LL-0047 method). Once the hash mismatch localised it to
  one column, a direct scan named the defect immediately rather than requiring
  a bisect:

  ```sql
  select id from t where col like '%''''%'          -- doubled apostrophe
     or other_col::text like '%''''%';
  ```

  The write was re-applied with correct escaping, re-hashed to confirm
  agreement, and every sibling row was scanned for the same pattern. The
  workflow was then changed so the content exists in **one** authored form: edit
  the `.sql` file, rebuild the local database from it, and hash-compare local
  against remote. That turns the comparison into a proof that the file and the
  remote agree, instead of a comparison between two independently hand-authored
  statements that can both be wrong.
- **Preventive Rule**: Never author the same content twice in two escaping
  contexts — generate the second form from the first, or make the file the only
  source and load from it. Where a hand-authored write into a nested literal is
  unavoidable, pair the per-column hash comparison with an explicit
  doubled-delimiter scan (`like '%''''%'` and the equivalent for whichever
  delimiter is nested); the hash tells you *that* something differs, the scan
  tells you *what*, which is the difference between a six-query bisect and a
  one-query answer. Never treat schema validation, JSON parse success, or
  successful response-model instantiation as evidence of content fidelity.
- **Similar Situations**: Any nested quoting context — JSON inside SQL, SQL
  inside a shell command, a regex inside JSON, YAML inside a heredoc, a quoted
  field inside CSV, or anything passed through a templating layer into a
  string literal. Note this is a genuinely different cause from LL-0047 and its
  preventive rule does not cover this case: LL-0047 says to have a script move
  the bytes rather than an agent, but here the corruption was already baked
  into the hand-authored statement, so a script would have faithfully
  transported the wrong bytes. The two entries share only a detection method —
  per-column byte comparison — which is what makes that method worth running
  after *every* content write, regardless of how the content got there.

### LL-0049 — Decorative text inside a control silently joins its accessible name, and optional-prop fixtures hide it

- **Root Cause**: A new `CategoryRow` component rendered a category toggle as a
  `<button>` containing the category name plus, when the category had a lesson,
  a muted second line with the lesson title. The title span carried no
  `aria-hidden`, so it contributed to the button's computed accessible name.
  The toggle announced as "Management of Care Delegation, Assignment, and
  Prioritization", and because the adjacent study link already carried
  `aria-label="Study: <title>"`, a screen-reader user heard every lesson title
  twice per row.
- **Why It Happened**: Two independent failures had to line up. First, the
  accessible name of a control is the concatenated text of its *entire*
  subtree — visual hierarchy (a smaller, muted, secondary line) carries no
  semantic weight at all, so text that reads as decoration to a sighted
  reviewer reads as part of the label to assistive tech. Second — and this is
  the transferable part — the whole test suite was blind to it. Eight
  component tests and five integration tests existed, several querying the
  toggle by exact category name via `getByRole("button", { name: category })`,
  which is precisely the assertion that would have caught this. Every one of
  them passed because the fixtures they used omitted the optional `lesson`
  prop, so the extra text never rendered in any test that could have detected
  it. In production all 8 categories have a lesson, so the defect was present
  on 100% of real rows and 0% of tested rows.
- **Solution**: Marked the second-line span `aria-hidden` — the title stays
  visually present (it is the discoverability payload) and still reaches
  assistive tech through the sibling link's `aria-label`. Then added a test
  that renders the row *with* the optional prop populated and asserts the
  toggle's accessible name is exactly the category string, verified red before
  the fix and green after.
- **Preventive Rule**: When a control's subtree contains text that is not part
  of its label, mark it `aria-hidden` and prove it with a test that asserts the
  exact accessible name (`getByRole(role, { name: "..." })` with a string, not
  a regex — a substring or regex matcher passes right through this bug). More
  generally: **for any component with an optional prop that renders additional
  content, at least one test must exercise the populated branch against every
  assertion that could be perturbed by it.** A fixture that omits an optional
  prop is not a neutral default — it is an untested configuration, and when the
  omitted case is the one that never occurs in production, the suite is testing
  a shape that does not ship.
- **Similar Situations**: Any accessible-name computation — buttons and links
  containing badges, counts, timestamps, "new" pills, icon labels, helper text,
  or truncated previews. The fixture half generalizes past a11y entirely: any
  partial-fixture cast (`as unknown as T` with fields omitted) creates a gap
  between what tests render and what production renders, and defects live
  exactly in that gap. Related to the NCLEX project's own finding that a
  fixture's `as unknown as Question` cast omitting required fields masked
  unguarded `.length` reads — same root shape, different symptom.

---

### LL-0050 — Mocking an integration function wholesale leaves its own response-parsing body with zero coverage

- **Root Cause**: A Clerk API wrapper, `find_pending_invitation`, read the
  `GET /invitations` response as a `{"data": [...]}` envelope, but the API
  answers with a bare JSON array. Every real call raised
  `AttributeError: 'list' object has no attribute 'get'`. The sibling function
  `list_users` in the same module already did `for u in response.json():` —
  i.e. the correct shape was documented right there in the codebase, ten lines
  below the bug.
- **Why It Happened**: The project had an explicit, sensible convention that
  every third-party-API function is mocked at the boundary and never really
  called in CI. That keeps tests fast and hermetic, but it means the mocked
  function's *own body* — the HTTP call and, critically, the response
  **parsing** — is the one piece of code no test ever executes. Callers'
  tests all `monkeypatch.setattr(clerk_admin, "find_pending_invitation", ...)`,
  so a 100%-broken adapter passed a 618-test suite. The shape assumption was
  never validated against a real response, only against a hand-written mock
  that encoded the *same wrong belief* as the code.
- **Solution**: Fixed the parse to accept both shapes
  (`payload.get("data", []) if isinstance(payload, dict) else payload`) and
  added tests that exercise the function *body* against a fake `httpx`
  response object, covering the bare array, the envelope, case-insensitive
  matching, no-match, empty, and non-2xx. Verified red before / green after.
- **Preventive Rule**: **Mock at the transport, not at the function, for at
  least one test per integration wrapper.** If you monkeypatch the whole
  function everywhere, add a companion test that stubs the HTTP client
  (`httpx.get`, `fetch`, the SDK's session) and drives the wrapper's real
  parsing code with a payload **copied from an actual recorded response**, not
  invented. A mock you wrote from the same mental model as the code under test
  cannot falsify that model. When several wrappers hit the same API, make their
  response handling visibly consistent — a neighbouring function using a
  different shape for the same vendor is a defect signal, not a style choice.
- **Similar Situations**: Any API-client/adapter/serializer layer —
  payment providers, auth vendors, LLM SDKs, webhooks, storage clients.
  Especially acute when a vendor returns different envelope shapes per
  endpoint or changes them across API versions. Related to LL-0049's fixture
  point: a mock that omits or misstates reality is an untested configuration,
  and defects live exactly in that gap.

---

### LL-0051 — A platform error page carries no CORS headers, so the browser hides the real status behind a generic network error

- **Root Cause**: A 500 from a FastAPI app on Vercel is rendered by the
  *platform*, not the app, so it does not pass through the app's
  `CORSMiddleware` and ships without `access-control-allow-origin`. The
  browser therefore blocked the response before JS could read it, `fetch()`
  rejected with `TypeError: Failed to fetch`, and the UI's catch-all
  translated that into "Can't reach the server — check your connection."
  The actual event was a server-side `AttributeError`.
- **Why It Happened**: The app's *own* error responses (401/409/502) do carry
  CORS headers, because those are generated inside the app and pass through
  the middleware. That makes the correct-looking inference — "error responses
  are fine, so this must be connectivity or CORS" — and sends the
  investigation to the wrong layer. CORS preflight also passed cleanly, which
  reinforced the wrong conclusion.
- **Solution**: Stopped trusting the client-side message and read the
  platform's own runtime logs, which contained the full traceback with the
  exact file and line. Root cause found in one step after that.
- **Preventive Rule**: When a browser reports a *network-level* failure
  (`Failed to fetch`, `Load failed`, status 0) against an endpoint whose
  preflight succeeds and whose sibling endpoints work, **treat it as an
  unhandled server exception until the server logs say otherwise** — do not
  debug it as CORS or connectivity. Go to the server/platform logs first; the
  client cannot see this class of error by construction. Corollary for
  diagnosis: a client error string that is a *catch-all* (`catch { throw
  genericMessage }`) carries no information about the actual failure — treat
  it as "unknown", never as evidence.
- **Similar Situations**: Any serverless/edge-hosted API behind a
  browser client — Vercel, Netlify, Lambda+API Gateway, Cloudflare Workers.
  Also gateway timeouts (504) and OOM kills, which fail the same way. The same
  masking happens with any framework where the platform, not the app, renders
  the failure page.

---

### LL-0052 — A per-item `try/except: log` in a batch loop turns total failure into silence

- **Root Cause**: A daily cron's invite-reminder loop wrapped each row in
  `try/except Exception: logger.exception(...)`. When a shared dependency
  broke for *every* row, the job still returned HTTP 200 with a normal-looking
  result body — `invite_reminder_sent: 0` — and no alert fired. A feature that
  had shipped the day before was sending zero emails, and the only evidence
  was buried in per-row log lines nobody was reading.
- **Why It Happened**: Per-item isolation is the right instinct — one bad row
  shouldn't kill a batch. But it silently conflates two very different
  outcomes: "1 of 200 rows failed" (fine, keep going) and "200 of 200 rows
  failed" (the feature is dead). A zero counter is indistinguishable from
  "nothing was due today", which is also the *normal* state for this job, so
  even a human reading the output would not flinch.
- **Preventive Rule**: In any batch/cron loop with per-item exception
  handling, **count failures alongside successes and return both**, then make
  a total or near-total failure rate loud: raise, alert, or at minimum return
  a distinct status when `failed > 0 && succeeded == 0`. Never let "attempted
  N, succeeded 0, failed N" and "attempted 0" render identically. When a
  scheduled job's success metric can legitimately be zero, that metric alone
  cannot be used to verify the feature works — pair it with an explicit
  failure count.
- **Similar Situations**: Cron jobs, queue consumers, bulk importers,
  notification fan-outs, webhook retry loops, migration scripts. Also
  applies to log-and-swallow around optional side effects (search indexing,
  analytics, cache warming) where the swallowed error means the feature is
  entirely inert. Related to the NCLEX project's earlier finding that
  indexing failures were log-and-swallowed, so a 200 response proved nothing
  and the index had to be counted directly.

---

### LL-0053 — Automated browser tools suppress native `confirm()`, so a confirm-gated action silently no-ops

- **Root Cause**: An admin button gated behind `window.confirm()` appeared
  completely dead when driven through an agent-controlled browser pane. The
  automation layer disables native JS dialogs by design and returns `false` to
  the page, so the guarded branch never ran and no request was ever sent. The
  console said so explicitly — `Page dialog suppressed (confirm): ... confirm()
  returned false to the page` — but only when read directly.
- **Why It Happened**: The failure is indistinguishable from a broken
  handler: no error, no network activity, no UI change. It also survives the
  obvious workaround of "have the human click it", because a human clicking
  *inside the same automated pane* hits the same suppression — the property
  belongs to the browser context, not to who moved the mouse. That cost
  several rounds of misattributed debugging.
- **Preventive Rule**: When a UI action produces **no network request at
  all**, read the browser console before touching application code, and check
  for dialog suppression specifically if the control is gated by
  `confirm`/`alert`/`prompt`. For verification that must exercise a
  confirm-gated path, drive it from a genuinely separate browser process, or
  test the endpoint directly. Do not "fix" this by patching `window.confirm`
  in an automated session — that defeats a deliberate safety guard and, worse,
  would have hidden the real server-side bug underneath it.
- **Similar Situations**: Any agent/CI-driven browser (Playwright, Puppeteer,
  Selenium, agent browser panes) meeting `confirm`, `alert`, `prompt`,
  `beforeunload`, file pickers, permission prompts, or basic-auth dialogs.
  Generalises to a debugging rule: **"no request was sent" and "the request
  failed" are different bugs living in different layers** — distinguish them
  before forming a hypothesis.

### LL-0054 — A local `.env` makes the local suite more permissive than CI's, so "green locally" can mean "never ran in CI"

- **Root Cause**: The seven tests written to close LL-0050 — the ones that
  finally exercise a Clerk wrapper's real parsing body instead of mocking the
  function wholesale — failed in CI on every push with
  `503: Clerk admin API is not configured: set CLERK_SECRET_KEY`. Running the
  real body reaches the module's `_headers()` credential guard, which raises
  when the key is unset. A developer machine has a `.env` supplying one; the
  CI job sets exactly three env vars and nothing else. So the fix for LL-0050
  was green locally and red in CI **from the moment it landed**, and stayed
  red across three consecutive pushes because the local signal said green and
  nobody opened the Actions tab.
- **Why It Happened**: Two compounding causes. First, the *direct* one: the
  suite's convention of mocking integrations at the function boundary meant no
  existing test had ever reached a credential guard, so nothing established
  the habit of pinning secrets in-test. The new tests were the first to run a
  real body, and thus the first to need one. Second, the *systemic* one:
  nobody had ever compared the local environment to CI's. Locally `.env`
  supplied **every** secret — Clerk, Qdrant, Voyage, Resend, cron — while CI
  supplied three database vars. The two suites had silently diverged into
  different suites, and the more permissive one was the one being used to
  decide whether to push. Blanking an env var did not reproduce it either:
  pydantic-settings ran with `env_ignore_empty=True` and an `.env` path
  anchored to the repo root, so `SECRET="" pytest` still loaded the real
  value — the obvious reproduction attempt produced a false all-clear.
- **Solution**: Pinned a fake key via an autouse fixture in the affected
  module rather than adding a CI secret, so it depends on nothing outside the
  repo. Then fixed the class of bug rather than the instance: an autouse
  fixture in the root `conftest` now resets every setting CI does not provide
  back to its class default, so a local run can only pass if it would pass in
  CI. A companion test parses the workflow file and asserts the "CI provides
  these" list matches the job's actual `env:` block, so adding a CI env var
  without updating the list fails loudly instead of silently restoring the
  permissive run. Proven rather than assumed: disabling the module fixture now
  makes those seven tests fail *locally*, where before they failed only in CI.
- **Preventive Rule**: **Make the local test environment provably equal to
  CI's, and check CI after pushing — a green local run is evidence about your
  machine, not about the build.** Concretely: enumerate what CI provides,
  reset everything else to defaults in a root-level autouse fixture, and tie
  that enumeration to the CI config with a test so the two cannot drift. Any
  test that genuinely needs a credential pins it itself. Never conclude a
  secret-dependent test is environment-independent without simulating the
  absence — and verify your simulation actually took effect, because settings
  libraries routinely ignore empty env vars in favour of a file.
- **Similar Situations**: Any repo where developers keep a `.env` that CI
  lacks — which is most of them. Especially dangerous when the local values
  are *production* credentials: here `QDRANT_URL`, `VOYAGE_API_KEY` and
  `RESEND_API_KEY` were live, so the same divergence that hid a CI failure
  also let unmocked tests reach production services (the cause behind an
  earlier incident where the suite wrote fixtures to the live vector index).
  The same shape appears with feature flags defaulted on locally, seeded
  fixture data absent in CI, and locale or timezone set only on the laptop
  (cf. LL-0039). Direct sequel to LL-0050: closing a coverage gap moved the
  code into a new environment, and the new environment had its own gap.

---

### LL-0055 — When one dependency has no local equivalent, the whole test suite quietly runs against production

- **Root Cause**: Every `pytest` run wrote test-fixture points into the
  **production** vector index. `SUPABASE_URL` pointed at a local stack, but
  `QDRANT_URL` pointed at Qdrant Cloud — there is no local Qdrant — so eight
  publish tests that never patched the indexer called the real embeddings API
  and the real production collection. Each run deleted its fixture rows from
  the local database but nothing de-indexed the remote point, so every run
  left an orphan behind in live retrieval.
- **Why It Happened**: "Local development environment" was treated as a single
  binary state rather than a per-dependency property. Most of the stack had a
  local equivalent, so the mental model was "tests run locally"; the one
  managed service without a local option silently broke that. Two things then
  hid it: the publish route **log-and-swallows** indexing errors, so the tests
  passed whether the write succeeded or failed (cf. LL-0052), and the orphaned
  data lived in a system nobody inspected during normal work. It surfaced only
  when someone scrolled the live index for an unrelated reason and found a
  fixture record with no matching row.
- **Solution**: An autouse fixture in the root `conftest` neutralises all four
  index/deindex entry points for the whole suite; tests that assert indexing
  patch those names themselves and still win, and the module that tests the
  indexer is exempt because it stubs one layer lower. The entry points are
  kept in a single named list so a new one cannot be added without the guard
  covering it. An equivalent guard already existed for ops scripts — written
  after a backfill deleted published content from the same live index — but
  nobody had asked whether the test suite needed one too.
- **Preventive Rule**: **Enumerate every external dependency and ask, per
  dependency, "what does a test run hit?" — not "am I running locally?"** Any
  dependency with no local emulator needs an explicit suite-wide block, not a
  per-test convention, because the failure mode is silent and cumulative.
  Corollary: **deleting the artefact without fixing its producer is theatre** —
  the next run recreates it, so always find what created a stray record before
  calling the cleanup done. And when you build a safety guard for one entry
  point into live infrastructure (ops scripts), immediately ask which *other*
  entry points (test suite, local CLI, notebooks) share the same exposure.
- **Similar Situations**: Any managed-only service — hosted search, vector
  stores, payment sandboxes with a single shared tenant, email senders,
  analytics, feature-flag services, LLM APIs. Especially where the client
  log-and-swallows errors, since that removes the loudest signal that a test
  is talking to something real. Directly related to LL-0054: the same
  local/CI environment asymmetry, seen from the other side.

---

### LL-0056 — A suppression-only test suite passes under an inverted comparison, so escalation is never actually tested

- **Root Cause**: A day2/day4/day7 email escalation deduped by checking each
  stage's own identifier, so a long-overdue user got "day7" on the first run,
  then wrongly fell back to "day4" on the next and "day2" after that —
  descending urgency, exactly backwards. Fixed to a rank check: find the
  single highest currently-due stage and fire it only if nothing at least that
  urgent was already sent. Then the review found the **fix itself was
  untested**: no test anywhere asserted either function ever returns the middle
  stage, and a mutation (flip `<=` to `>=`) proved every existing test still
  passed while silently truncating both sequences at the first stage forever.
- **Why It Happened**: The tests were all written from the "don't spam people"
  worry, which is the *suppression* property: stage N was sent, does stage N
  correctly not re-fire? Every one of those passes under an inverted
  comparison, because a mutation that suppresses too much still suppresses.
  The complementary property — stage N was sent, does stage N+1 correctly fire
  later? — is the *escalation* path, and nothing exercised it. The two read as
  the same feature in prose ("send the right reminder at the right time") but
  are separate behaviours with opposite failure directions, and a test suite
  written from one worry is blind to the other.
- **Solution**: Added explicit escalation tests for both functions, each
  **independently mutation-verified by the reviewer** — not merely trusted from
  the fixer's report — to be the sole detector of its mutation across a 60+
  test blast radius. Note the original defect was a bug in the written *plan*,
  not an implementer error, and was caught by a task reviewer mid-implementation;
  the plan document was corrected before the fix was dispatched.
- **Preventive Rule**: **For any state machine or staged sequence, test that it
  advances, not only that it doesn't repeat.** Write the pair explicitly: "N
  sent → N does not re-fire" *and* "N sent → N+1 fires when due". When a fix
  changes a comparison operator, mutate it and confirm at least one test fails;
  if none does, the fix is unverified regardless of how many tests are green.
  Generalises beyond sequences: **a suite written from a single worry only
  tests one direction of the property**, so name the opposite failure
  ("fires too little" vs "fires too much") and check you have a test for each.
- **Similar Situations**: Retry/backoff ladders, dunning and reminder emails,
  onboarding drip sequences, escalation policies in on-call, rate limiters,
  cache invalidation, feature-flag rollout stages, order/subscription state
  machines. Any place where "did nothing" is a plausible-looking outcome — see
  LL-0052 for the same shape at the batch-loop level.

---

### LL-0057 — Fixing a generator is not enough if its own input was already corrupted by the earlier run

- **Root Cause**: A seed-generation script built its INSERT from a
  hand-transcribed list of table columns that omitted two real ones, so a
  database reset seeded 200 rows with NULL in both — silently, because an
  INSERT naming a subset of columns just applies the defaults. A whole-bank
  per-column parity check caught it. The column list was then fixed — and the
  very first regeneration **reproduced the identical mismatch**, because the
  generator reads from the local database to build its values, and local had
  already been overwritten with the NULLs by the bad reset. Regenerating from
  a corrupted source faithfully re-encodes the corruption.
- **Why It Happened**: The mental model was "generator produces artefact", so
  fixing the generator felt like fixing the problem. But the pipeline was
  really source → generator → artefact → **source**, a loop the earlier run had
  already poisoned. The second failure is more instructive than the first: the
  fix was correct and still produced a wrong result, which is exactly the
  situation that erodes trust in the fix rather than in the input. The original
  slip was mundane — a column list written from memory instead of queried.
- **Solution**: Two steps, in order: restore the input from a known-good source
  (the original draft data, re-upserted) to repair the corruption, **then**
  regenerate from the now-correct source. Plus a standing guard: the generator
  now asserts its column list against a live `information_schema` query as an
  exact set equality — not a count — before writing anything, so a schema change
  or a mistyped list fails loudly instead of defaulting a column to NULL.
- **Preventive Rule**: **When a generated artefact is wrong, fixing the
  generator is necessary but not sufficient — check whether the generator's
  input was already corrupted by the same bug's earlier run, and repair the
  input first.** Ask "what does this read from, and did the broken version
  write there?" before regenerating. Separately: **never hand-transcribe a
  schema.** Derive column lists from the live schema and assert set equality,
  because a subset is silently valid in SQL and produces defaults rather than
  an error.
- **Similar Situations**: Code generators reading a database or a previous
  build output, migration/backfill scripts, snapshot tests regenerated with
  `--update` after a real regression, lockfiles, denormalised caches rebuilt
  from a table the bug already wrote to, ETL that re-ingests its own exports,
  and any "just regenerate it" recovery step. Related to LL-0044 (a backfill
  must replay every field the real flow sets) and LL-0036 (prove a
  reconstruction against its source, don't eyeball it).

---

### LL-0058 — Randomising presentation order invalidates every stored reference to a position, including in text a downstream system generates

- **Root Cause**: Exam options are shuffled per session so their displayed
  letters differ from stored order. An LLM tutor answering "why is my answer
  wrong?" was handed the stored options and referred to them by their stored
  letters, so it told students about "option B" when the student had seen that
  content as D. It also never learned which option the student actually chose.
  The same class of defect existed in authored content: rationales referencing
  "option B" and items whose stored order *was* the answer order.
- **Why It Happened**: Shuffling was introduced as a display concern, and
  display concerns are assumed not to reach back into stored data. But a letter
  is an identifier the content refers to, so randomising the mapping between
  identifier and position retroactively falsified every stored sentence that
  named one — including sentences generated later by a model reading the stored
  form. Nothing failed loudly: the tutor's answer was fluent and confidently
  wrong, and the authored rationales were still grammatical.
- **Solution**: Pass the session's actual display order (and the student's
  chosen option) to the consumer, and re-letter options, correct answer, and
  rationale by display position before use; ignore a display order that isn't a
  valid permutation, so nothing unverifiable can steer generated content. Pin
  the contract with tests on both sides of the wire, and verify live that a
  stored key rendered at a different position is described by its *displayed*
  letter. In the authoring rules, forbid referring to options by letter at all —
  quote or describe the option's content instead.
- **Preventive Rule**: **If presentation order is randomised, positional labels
  are not stable identifiers — ban them from stored content and pass the live
  ordering to anything that renders or reasons about it.** Write rationales,
  explanations, and prompts to reference *content*, not position. When adding a
  shuffle to an existing system, grep for every stored string that names a
  position and treat each as a defect. And whenever an LLM consumes your stored
  data, remember it will faithfully reproduce whatever stale identifiers you
  hand it — with total fluency and no error signal.
- **Similar Situations**: A/B-ordered survey questions, randomised quiz and
  poll options, shuffled search or recommendation slates referenced in
  copy/analytics, "the third card" in onboarding text, screen-reader labels
  derived from index, and any RAG or prompt context assembled from stored
  records whose display form differs from storage. The general shape: an
  identifier that is only valid within one rendering, leaking into a durable
  artefact.

---

### LL-0059 — A sync tool that carries only part of the state a workflow mutates reports success while silently dropping the rest

- **Root Cause**: A script reconciles review state from a production database
  into a checked-in seed file. Its per-table spec declared which columns to
  *carry*. For one table it carried three columns (audit trail, verified-source
  flags, lifecycle stage); for the other it carried only the audit trail. A
  review of 200 items then published all of them — which moves the stage and
  flips the verified flags — and the tool reported "200 rows topped up" and
  wrote the audit trail alone. A database reset would have replayed every one
  of those rows back to unreviewed with unverified sources, while the audit
  trail claimed a human had approved them.
- **Why It Happened**: The narrow spec had never been wrong *before*, because
  every prior run of that path happened to be an append-only action that moves
  no state. The tool was therefore "proven in production" against exactly the
  one mutation that could not expose the gap. The identical hole had been found
  and fixed in the sibling table three days earlier, and the fix was applied
  where the bug was observed rather than to the shared concept — leaving an
  asymmetry between two specs of the same shape, which reads as deliberate
  design rather than an unfinished fix.
- **Solution**: Give both tables the same carry set and widen the guarded
  content columns; add regression tests per table, each verified to fail
  against the pre-fix spec so they are not vacuous. Then prove the whole thing
  end to end rather than trusting the tool's own report: replay the seed from
  scratch and hash every column on both sides, which is what showed the state
  columns converging and the content columns already matching.
- **Preventive Rule**: **Derive a sync tool's carry set from the full set of
  columns the workflow can mutate, not from the mutations you have seen.** Ask
  "what does the *widest* action on this path change?" and carry all of it. A
  tool whose per-entity configs are the same shape but different contents is
  asserting a real difference — if you cannot state why, it is an unfinished
  fix, not a design. And when a bug is found in one entity handled by shared
  machinery, immediately check every sibling before closing it.
- **Similar Situations**: Any partial-state reconciler — env/config sync
  between environments, cache warmers, search-index updaters, CRM or billing
  sync jobs, fixture regenerators, replication filters, and ETL column
  allowlists. The general shape: a tool that reports on the subset it knows
  about and stays silent about everything it was never told to look at, so its
  success message is scoped to its own blind spot.

---

### LL-0060 — A capped read fails by succeeding, and a completion message counted from the truncated set will confirm it

- **Root Cause**: A backfill that embeds every published record into a vector
  index read its source with an unpaginated, unordered `select`. The data layer
  (PostgREST) caps such a read at 1000 rows and returns 200 OK with no warning,
  so the job processed 1000 of 1635 records and printed "Done: all 3015 claims
  indexed." Because no `order` was specified, *which* 1000 rows came back was
  arbitrary — so the missing records were scattered across every id prefix
  rather than forming a contiguous tail, and the damage read like sporadic
  per-item failures in a downstream service rather than a bad query.
- **Why It Happened**: The job was written and validated when the table was
  under the cap, so it was correct at the time and silently became wrong as the
  data grew — the same growth blind spot as LL-0027, but inverted: there the
  request grew until it broke loudly, here the response quietly shrank. The
  completion message made this durable: it counted what the job *processed*,
  which by construction can never reveal what it failed to read. Two separate
  investigations had already accepted a plausible wrong cause (rate-limit
  failures in an earlier run) because the numbers were self-consistent.
- **Solution**: Paginate with an explicit `order` and a fixed page size, looping
  until a short page. Add tests whose fake client *models the 1000-row cap*
  rather than the client's happy path — modelling the cap is what makes them
  fail against the old code with a meaningful assertion instead of erroring on
  a missing attribute. Verify the fix by the count the job *collected* against
  the count the source *holds*, not by the job's exit status.
- **Preventive Rule**: **Never read "everything" in one call from a paginated
  API; and make every bulk job compare what it processed against what the
  source says exists, failing loudly on a mismatch.** A count printed from the
  set you just walked is not evidence of completeness. When a bulk job's output
  looks partial, suspect a truncated *read* before blaming the write path — and
  when a data-layer read has no `order`, treat the returned subset as arbitrary,
  which makes "scattered, random-looking gaps" a symptom of truncation, not
  evidence against it.
- **Similar Situations**: PostgREST/Supabase default limits, DynamoDB scan
  pagination, S3 `ListObjects` truncation, Elasticsearch's 10k window, GitHub
  and Stripe API page defaults, LDAP size limits, BigQuery/Sheets export caps,
  and any "reindex/backfill/migrate everything" job whose success line is a
  count of its own work.

---

### LL-0061 — A credential pointing at the wrong environment returns a subset, not an error, so the read looks like data loss

- **Root Cause**: A lookup that resolves user records ran with a local
  credential for the provider's **development** instance, while the data being
  joined against had been produced, in part, by the **production** instance.
  The call returned HTTP 200 with 5 of ~15 records. Four ids were looked up;
  one resolved and three did not. Nothing errored, no warning was emitted, and
  the natural reading of the result — "those three records were deleted" — was
  wrong in a way that would have shaped the next decision.
- **Why It Happened**: The system had migrated to a production instance on a
  known date, and identifiers minted before and after that date belong to
  *different* instances. No single credential can resolve the full history, and
  each environment answers confidently about the half it owns. The local config
  had never been repointed because every prior task happened to touch records
  from one side of the cutover only. The first instinct was to suspect
  pagination — the same failure shape as LL-0060 — but an explicit high limit
  was already set, which cost a detour before the environment split was
  considered.
- **Solution**: Confirm which instance a credential addresses *before* trusting
  a lookup that spans a migration boundary — the cheapest signal is a total
  count against a known expected magnitude (5 vs ~15), not an inspection of the
  key itself. Where the missing records could not be resolved directly,
  existence was proven **indirectly through side-effect history**: rows in a
  delivery log showed the production system had successfully resolved those
  same ids days earlier, which established reachability without ever holding
  the production credential.
- **Preventive Rule**: **Treat a short result from an environment-scoped
  credential as a wrong-environment signal until proven otherwise, and never
  conclude "the record is gone" from a lookup you have not confirmed is pointed
  at the environment that owns it.** When a migration splits identifiers across
  two instances, write down which side of the cutover each cohort lives on —
  that fact is invisible in the data and expensive to rediscover. Prefer
  proving existence from a system's own side-effect log over acquiring a
  production credential to answer the question directly.
- **Similar Situations**: Clerk/Auth0/Firebase dev-vs-prod tenants, Stripe test
  vs live keys, sandbox vs production payment and e-signature APIs, a staging
  database URL left in a local env file, multi-region accounts where a regional
  endpoint answers only for its own region, and any post-migration system where
  old and new identifiers coexist in one table.

### LL-0062 — A service that reports "healthy" before it has finished migrating will make your verification roll back a healthy system

- **Root Cause**: An upgrade script waited for the service's health endpoint to
  return 200, then immediately read database state to compare against a
  pre-upgrade baseline. The service binds its health port *before* schema
  migrations begin — six milliseconds before, per its own startup log — so the
  comparison ran against a database another process was actively migrating.
  Counters that legitimately change during startup (webhook re-registration for
  active workflows) would mismatch, and a mismatch triggers automatic rollback:
  stop the service, move the data directory aside, restore from backup.
- **Why It Happened**: "Healthy" and "ready" answer different questions, and
  most services expose the first by default. Liveness asks "is the process up";
  readiness asks "can it serve correctly". The upgrade needed the second and
  asked the first. This is invisible in testing, where there are few migrations
  and no active work to re-register, so the race only appears against a real
  system — and it appears on the *success* path, meaning the least-tested code
  (rollback) becomes the expected outcome of a normal upgrade.
- **Solution**: Gate verification on a readiness endpoint that explicitly means
  "connected and migrated", and require both liveness and readiness rather than
  substituting one for the other. Re-read volatile counters a bounded number of
  times before declaring failure, so a single transient read cannot trigger a
  destructive rollback. Verify the readiness response *body*, not just its
  status code: many web applications return HTTP 200 with an HTML page for any
  unknown path, so a mistyped readiness URL fails open and silently reinstates
  the whole race with no log line.
- **Preventive Rule**: **Never treat a liveness signal as permission to read
  state you intend to compare.** Ask what the endpoint actually asserts, not
  what its name suggests. When a check's failure mode is destructive, make it
  retry before it fires, and make it fail closed when its own probe looks
  wrong — a health gate that accepts any 200 is not a gate.
- **Similar Situations**: Kubernetes liveness vs readiness probes, connection
  pools that accept before warming, app servers that bind before running
  migrations (Rails, Django, Keycloak, n8n), load balancers adding a backend on
  a bare TCP check, post-deploy smoke tests racing an async warmup, and any
  "wait for healthy then assert" step in a deployment pipeline.

### LL-0063 — An automated retention policy that matches by prefix will eventually delete something a human made

- **Root Cause**: A backup routine globbed `<prefix>-*`, sorted lexically, and
  deleted everything beyond a retention count. During an incident a human had
  created a rescue backup named `<prefix>-2026-08-10-pre-2.33.7`, while the
  script's own format was `<prefix>-20260810-215728`. Because `-` sorts below
  `0`, the hand-named directory sorted as the *oldest* entry and would have been
  the first thing deleted — destroying the only pre-incident restore point as a
  side effect of a routine upgrade.
- **Why It Happened**: The glob expressed "things that look like my backups"
  while the intent was "things I created". Those sets differ only once a human
  puts something in the same directory — which is exactly what happens during an
  incident, under time pressure, when someone wants a rescue copy somewhere
  obvious. A second instance of the same confusion: disposable copies produced
  by dry runs were competing for retention slots with real verified backups, so
  merely running the test suite evicted genuine restore points.
- **Solution**: Restrict automated deletion to names matching the exact format
  the tool itself emits, and treat everything else as operator-owned and
  untouchable. Partition automated artifacts by kind so throwaway copies and
  irreplaceable ones never share a retention budget.
- **Preventive Rule**: **Automated cleanup deletes only what it can prove it
  created.** Match the generator's own format exactly, never a loose prefix, and
  never let disposable artifacts compete with irreplaceable ones for the same
  slots. Put the hazard in a comment beside the filter — otherwise a later
  reader will "simplify" it back to a glob.
- **Similar Situations**: Log rotation over a shared directory, snapshot and AMI
  pruning by name tag, cache eviction that catches pinned entries, `docker
  system prune` removing hand-tagged images, tmp-file reapers, and CI artifact
  retention deleting a manually uploaded diagnostic bundle.

### LL-0064 — "It was stopped a moment ago" is not "it is stopped now": re-check external state in the step that acts on it

- **Root Cause**: A script stopped a container, then several steps later copied
  its database file. In between, the container restarted itself — a restart
  policy of `unless-stopped` brings containers back when the daemon returns, and
  the daemon had just been upgraded. The copy therefore ran against a database
  with a live writer attached, producing a torn backup that nothing downstream
  would have flagged.
- **Why It Happened**: Classic check-then-act. The code reads as sequential and
  safe, and the gap was only a few statements. But the state being checked
  belongs to another system with its own opinions about when to run, so the size
  of the interval is not the risk — the existence of one is. A related defect
  sat in the state helper itself: it inferred "not running" from *any* failure
  to query, so a daemon hiccup or a permissions error read as "safe to proceed".
- **Solution**: Verify the precondition immediately before the operation it
  protects, in the same step, not earlier in the sequence. Make the state query
  distinguish running / stopped / *unknown*, and refuse on unknown — the same
  standard already applied to integrity checks elsewhere in the same script.
- **Preventive Rule**: **A precondition checked earlier than the action it
  guards is documentation, not a guard.** Re-verify at the point of use, and
  make "cannot determine" refuse rather than proceed. A guard that fails open is
  worse than none, because it is trusted.
- **Similar Situations**: TOCTOU file races, `kill` after a PID lookup, lock
  files, acting on a lease long after acquiring it, deleting a resource after
  listing it, systemd `Restart=always`, Kubernetes controllers recreating a pod
  you just deleted, and autoscalers replacing an instance mid-maintenance.

### LL-0065 — Inspecting a system read-write perturbs it, and the perturbation looks like the bug

- **Root Cause**: While diagnosing a possibly-corrupt database, the
  investigation opened it with a default (read-write) connection to run an
  integrity check. That triggered write-ahead-log recovery and created sidecar
  files, after which the same file reported *malformed* on one run and *clean*
  on the next. Time was lost chasing a corruption the diagnosis had itself
  partly manufactured, and two byte-identical copies produced contradictory
  results.
- **Why It Happened**: The default connection mode of most database clients is
  read-write, and "I am only running a PRAGMA" feels read-only. The write
  happens in the engine's recovery path, not in the statement being issued, so
  nothing in the visible command hints at mutation.
- **Solution**: Route every inspection through an explicitly read-only handle,
  isolated in a single helper so no future call site can forget. Work on copies
  for any destructive experiment. When results differ between identical inputs,
  suspect the observer before concluding the data is unstable.
- **Preventive Rule**: **Diagnosis must not be able to write.** Make read-only
  the mechanically enforced default for every inspection path rather than a
  habit at each call site, and treat non-reproducible results across identical
  inputs as evidence that the tooling is mutating state.
- **Similar Situations**: SQLite WAL recovery on open, `fsck` on a mounted
  filesystem, git commands that take index locks or refresh the index, GUI
  database clients that silently run migrations on connect, profilers that alter
  timing, and debuggers whose breakpoints change scheduling.

### LL-0066 — A test hook in production code becomes a silent-failure path in production

- **Root Cause**: To exercise a reporting branch, an environment-variable
  override was added to a production script so a test could inject synthetic
  data. Feeding that override malformed input made the script exit 0 with empty
  output — indistinguishable from a successful run with nothing to report, in a
  tool whose entire contract was that silence must be unambiguous. Nothing gated
  the hook to a test context, so a stray export in a cron environment would have
  fabricated the report.
- **Why It Happened**: The branch looked untestable end to end, so a backdoor
  felt pragmatic. It was not untestable: shadowing a real dependency on `PATH`
  so that detection genuinely failed exercised the same branch through the
  production path, and tested it better — the synthetic-data version never ran
  the detection logic at all, which was the part that could break.
- **Solution**: Delete the hook and test through the real path by controlling
  the environment the real code reads. Where a branch genuinely resists that,
  extract the logic into a sourceable function and unit-test it. An escape hatch
  in the shipped artifact is the last resort, not the first.
- **Preventive Rule**: **Never ship a code path that only tests use.** If a
  branch seems unreachable in test, control the input the real code reads —
  `PATH`, config, fixtures, clock — rather than adding an input only the test
  knows about. Distinguish this from legitimate configuration: an override that
  selects an *input* still runs all the real logic, while one that replaces a
  *computed result* skips the very logic under test.
- **Similar Situations**: `if (env === 'test')` branches in shipped code, mock
  injection points left in release builds, debug flags that skip authentication,
  feature toggles read from unvalidated environment variables, and test seams
  that bypass validation.

### LL-0067 — A platform's "testing" mode expires credentials on a timer, and the symptom looks like random breakage

- **Root Cause**: An OAuth integration stopped working roughly a week after each
  authorization, repeatedly. The provider's app had been left in *Testing*
  publishing status, where refresh tokens are expired after seven days by
  policy. Nothing failed at authorization time, nothing warned as the deadline
  approached, and the automation simply went quiet — the outage ran nine days
  before anyone noticed.
- **Why It Happened**: Development-mode defaults are built for building, not for
  running, and the restriction is a policy timer rather than a capability
  difference, so everything works perfectly right up until it does not. The
  timeline also actively misleads: measuring from the credential's *creation*
  date rather than its most recent *re-authorization* hid the seven-day period
  and led to the correct cause being investigated, wrongly ruled out, and only
  confirmed later by reading the provider console directly.
- **Solution**: Publish the app to production status — which is free, and is a
  separate thing from *verification*, the paid step needed only to remove an
  unverified-app warning or exceed a user cap — then re-authorize so new tokens
  are issued outside the timer. Read the console for the actual status rather
  than inferring it from symptoms.
- **Preventive Rule**: **Before debugging a periodic credential failure, read
  the provider's publishing/environment status, and date the timeline from the
  last re-authorization rather than from creation.** For anything running
  unattended, derive a liveness signal from the work itself — "no successful run
  in N hours" — because scanning error logs fails precisely when a broken
  integration stops logging at all.
- **Similar Situations**: Google OAuth apps in Testing mode, Slack and Meta apps
  in development mode, Apple sandbox certificates, Stripe test keys, short-lived
  or self-signed TLS certificates, service-account keys under rotation policy,
  and free-tier API tokens with periodic invalidation.

### LL-0068 — A read-only inspection tool can transform what it returns, manufacturing the exact defect you are investigating

- **Root Cause**: A delivered email appeared to have a corrupted one-click
  unsubscribe URL and a corrupted `<meta viewport>` tag: `width=device-width`
  arrived as `width<U+FFFD>vice-width`, and `&token=49d7fb…` as `&tokenId7fb…`.
  The source template was correct. The damage was a spurious quoted-printable
  decode applied to text that had already been decoded: the API returns body
  data with the transfer encoding *already removed* while still exposing the
  original `Content-Transfer-Encoding: quoted-printable` header, and the client
  honoured that header and decoded a second time. The mail on the wire was
  fine; the reader broke it.
- **Why It Happened**: The tool was read-only, so it was trusted implicitly —
  the failure mode considered was "the tool cannot see everything," never "the
  tool alters what it shows." The corruption was also *plausible*: it landed on
  exactly the bytes that mattered, and only where `=` was followed by two hex
  digits, so `charset=utf-8` and `initial-scale=1` survived. A defect that
  selective reads as a real encoding bug, not as an artifact.
- **Solution**: Read a **control artifact from an unrelated producer** through
  the same tool. An email from a different sender on a different provider, read
  the same way, showed the identical `IE=edge` → `IE<U+FFFD>ge` damage. One
  vendor cannot corrupt another vendor's output, so the reader was proved
  guilty without ever obtaining the raw bytes.
- **Preventive Rule**: **Before believing a defect that only one tool can see,
  make that tool read something it cannot possibly be responsible for.** If the
  defect appears there too, the tool is the bug. Prefer a path that returns raw
  bytes for any question about encoding or formatting, and treat any reader
  that decodes, re-encodes, prettifies, or normalises as a suspect — being
  read-only makes a tool safe, not honest.
- **Similar Situations**: Mail APIs that pre-decode bodies but keep the original
  CTE header, terminals and log viewers that interpret escape sequences,
  clipboard managers that convert smart quotes, JSON viewers that reformat
  numbers or reorder keys, screenshot pipelines that rescale or re-compress,
  diff tools that normalise line endings or whitespace, and browser devtools
  that show a parsed DOM rather than the served HTML.

### LL-0069 — A GET that mutates is unsafe the moment its URL travels in an email

- **Root Cause**: An unsubscribe endpoint opted the recipient out on `GET`,
  immediately after verifying a token in the query string. The link was
  embedded in student emails. Mail clients, link-preview generators and
  corporate security scanners routinely fetch URLs found in delivered mail, so
  any of them could have silently opted a recipient out of email they still
  wanted — with no user action and no trace distinguishing it from a real
  click.
- **Why It Happened**: The token made the URL feel safe, which conflated two
  different properties: the token provides *authentication* (only the intended
  recipient's link works), not *intent* (someone deliberately chose this). A
  one-click flow also makes mutating-on-GET feel like the whole point, since
  any confirmation step reads as friction.
- **Solution**: Make `GET` render a confirmation that `POST`s, and move every
  mutation to the `POST` handler. For email specifically this is what RFC 8058
  standardises: advertise the URL in `List-Unsubscribe` only alongside
  `List-Unsubscribe-Post: List-Unsubscribe=One-Click`, which tells compliant
  clients to POST and keeps scanners that merely GET harmless.
- **Preventive Rule**: **Authentication is not intent — a credential in a URL
  proves who, never whether.** Keep every state change behind a non-safe HTTP
  method, and assume any URL that reaches an inbox will be fetched by machines
  that never read it.
- **Similar Situations**: Email confirmation and verification links, "approve"
  and "decline" links in notification mail, magic-login links consumed by a
  scanner before the user clicks, one-click order or RSVP actions, GET-based
  logout or delete endpoints reachable by prefetch, and any webhook-style URL
  pasted into a chat client that unfurls links.

### LL-0070 — A keyword search is not an existence check: it finds what is *named*, not what is *implemented*

- **Root Cause**: Verifying "does X already exist here?" with a substring query
  (`grep X`, `... where col like '%X%'`) only matches artifacts that happen to
  use the word. Anything that implements, teaches, or performs X while naming it
  differently — or never naming it at all — returns zero hits and is reported as
  absent.
- **Why It Happened**: A large content bank was checked for duplicate subject
  matter before adding new entries. Two candidate topics came back clean on a
  `%keyword%` query and were approved as absent. Both were already present: one
  existing entry described the condition entirely through its observable signs
  and never used its name; another described the same phenomenon using a
  different clinical term. Duplicates were authored on the strength of the clean
  query, and were caught only because a later stage independently re-verified
  instead of trusting the earlier "verified absent" claim.
- **Solution**: Search the *behaviour* and the *shape*, not the label — the
  distinctive combination of attributes, the surrounding actions, the values
  involved — and then **read the candidate matches** rather than counting rows.
  Treat zero hits from a single name-based query as "no evidence", never as
  "evidence of absence".
- **Preventive Rule**: **A name-based query can prove presence but never
  absence.** Before acting on "no results", ask what the thing would look like
  if it were written by someone who used different vocabulary — then search for
  that. Prefer two orthogonal queries over one keyword, and read matches instead
  of trusting a count.
- **Similar Situations**: Checking whether a helper/util already exists before
  writing a new one (the existing one has a different name); dead-code and
  unused-dependency detection where usage is dynamic or aliased; "is this
  endpoint used anywhere?" against callers that build the path by concatenation;
  duplicate-bug triage in an issue tracker where the same defect is described in
  different words; license and secret scanning that greps for known markers;
  checking whether a migration or config change was already applied.

### LL-0071 — An agent's report that it did something is not evidence that it did

- **Root Cause**: Delegated work is judged by the worker's summary rather than
  by the artifact. The summary is written from intent and memory, so it is
  sincere and confident even when the work did not land, landed partially, or
  landed differently than described.
- **Why It Happened**: Across one multi-agent run, three distinct instances:
  (1) a worker reported it had "avoided" a specific existing item's framing, and
  had in fact reproduced that item's exact premise in a secondary field;
  (2) a worker killed mid-run by a quota limit left a final message describing
  the *next* step it was starting, which read as though the earlier steps were
  complete — inspection showed none of them had been written; (3) the
  orchestrator's own change-detection heuristic reported 15 modified units, of
  which 14 were wrong in both directions (13 false positives from a truncation
  artifact, plus one real change it missed entirely because that change did not
  touch the field being compared).
- **Solution**: After any delegated step, verify against the artifact itself —
  read the file, query the database, diff against a snapshot taken before the
  step. Take that baseline snapshot *before* delegating, so the comparison is
  exact rather than heuristic. Where the work has a machine-checkable property
  (a count, a schema, a hash), assert it.
- **Preventive Rule**: **Trust the artifact, never the report — including your
  own.** A status field, a completion notification, and a summary are all claims
  about the work, not the work. This applies with equal force to the
  orchestrator's own verification shortcuts: check that your check is right
  before you act on it.
- **Similar Situations**: CI jobs reported green that skipped the suite; a
  deploy tool reporting success while serving the previous build; "migration
  applied" flags that don't match the schema; a subprocess whose exit code is 0
  because the failure happened in a pipe; any background or long-running task
  that reports progress separately from its output; hand-offs between humans
  where "that's done" means "I finished my part of it".

### LL-0072 — Every unit can be individually correct while the *set* leaks a pattern, and replacing the pattern is not removing it

- **Root Cause**: Correctness is checked per unit, but exploitability is a
  property of the collection. A generator that makes each item defensibly right
  can still produce a set where some incidental attribute — position, length,
  ordering, which option is never chosen — predicts the answer without the
  content being read at all.
- **Why It Happened**: Three independent reviewers of one generated batch each
  found a different set-level regularity that no single-item check could see: an
  option class that was correct in none of nine items; the intended answer being
  the longest option in every item of a group; and a classification sequence
  that alternated perfectly across three items. Every individual item passed
  review. Worse, an earlier round had "fixed" a comparable bias by imposing a
  strict rotation — which was exactly as predictable as the bias it replaced,
  because it relocated the signal instead of destroying it.
- **Solution**: Add a set-level pass that asks "what property of this collection
  predicts the answer without reading it?" — distribution, position, length,
  ordering, and never-selected classes. When fixing, verify the *replacement*
  distribution is irregular rather than merely different: make the
  never-correct class genuinely correct sometimes **while leaving it present
  elsewhere**, and vary rank rather than just avoiding the extreme, or
  "second-longest" simply becomes the new tell.
- **Preventive Rule**: **Validate the batch, not only the item — and when you
  break a pattern, check you have not authored a new one.** Any fix that
  imposes a deterministic scheme (a rotation, an alternation, a fixed ratio) is
  a relocated signal, not a removed one.
- **Similar Situations**: Generated test fixtures where every "invalid" case
  fails the same field; seeded or sampled data whose ordering encodes the label;
  synthetic benchmarks where difficulty correlates with input length;
  randomised A/B assignment that is balanced but cyclic; ID or token generation
  that is unique per item yet sequential across the set; shuffled UI options
  whose correct choice is always the most detailed one.

### LL-0073 — A one-sided assertion cannot tell "restored" from "never changed"

- **Root Cause**: A rollback drill asserted that the live database's migration
  count matched its pre-upgrade value after rollback. That assertion passes
  identically whether the restore genuinely rolled a migrated database back, or
  nothing ever migrated in the first place. The test could not distinguish
  success from a no-op, so a green result carried far less information than it
  appeared to.
- **Why It Happened**: The assertion was written from the perspective of the
  desired end state ("the database is back where it started"), which is the
  natural way to write it. But the end state is shared by the success path and
  the do-nothing path. Nothing in the test observed the *intermediate* state
  that only the success path produces.
- **Solution**: Assert both sides of the transition. The drill now checks that
  the live database is back to 224 migrations **and** that the failed-upgrade
  copy set aside during rollback shows 230. Neither number alone rules out the
  no-op; the pair can only be produced by real migrations that were really
  rolled back. Relatedly, the same drill refuses to run at all when its target
  version equals the installed version — it prints `SCENARIO 2 SKIPPED` with a
  reason rather than executing a sequence of no-ops and reporting every
  assertion green.
- **Preventive Rule**: **For any test of a reversible operation, ask what the
  assertions would do if the operation had never run.** If they would still
  pass, the test proves nothing yet. Capture evidence of the intermediate state
  — the discarded artifact, the moved-aside directory, a counter observed at
  peak — not only the restored end state. And make a test that cannot
  meaningfully run say so loudly instead of passing quietly.
- **Similar Situations**: Undo/redo tests that only compare the final document
  to the original; idempotency tests where the second run's no-op is
  indistinguishable from the first run having failed; cache-invalidation tests
  asserting a fresh read matches the old value; transaction-rollback tests that
  only check the row is unchanged; migration up/down tests that never assert the
  schema actually changed in between; retry logic verified only by the eventual
  success; feature-flag rollbacks checked only against the pre-flag behaviour.

### LL-0074 — A reproduction is only evidence if your harness matches the user's

- **Root Cause**: An invite-funnel investigation stalled on a screen reading
  "Access restricted — sign ups are currently disabled", reproduced twice, on
  two separate invitation tickets. It was taken as proof that invited users
  could not sign up. It was an artifact of the tester: the browser already held
  a signed-in session, and an auth provider cannot create a second account
  inside one. In a private window the same ticket showed the sign-up form
  immediately. No real invitee had ever seen the error.
- **Why It Happened**: The reproduction was stable and repeatable, which reads
  as strong evidence — and normally is. But *stability only establishes that the
  harness is deterministic, not that it resembles production*. Every repeat used
  the same contaminated session, so repeating it added confidence without adding
  information. The same session also produced two lesser versions of the mistake:
  an automated browser was served an interactive bot challenge (real users get an
  invisible pass), and `curl` got a 403 from the same host — both treated,
  briefly, as signal about users.
- **Solution**: The disproof came from data, not from another reproduction. The
  provider's own records showed invitations created in the *same second* — seven
  accepted, three not — which is impossible if the flow is globally broken. That
  reframed the question from "why is it broken" to "what is different about my
  attempt", and the session was the only candidate. Re-run in a clean session; it
  worked.
- **Preventive Rule**: **Before trusting a reproduction, list every way your
  environment differs from the affected user's — session state, cookies,
  automation flags, IP reputation, admin privileges — and neutralise them.** For
  anything auth- or onboarding-related, reproduce in a private window by default;
  a signed-in tester is not a new user and cannot simulate one. And when a
  reproduction conflicts with production data, believe the data: the data covers
  many users, the reproduction covers one contaminated environment.
- **Similar Situations**: Testing a paywall or trial-expiry flow while holding an
  active subscription; checking a first-run or onboarding tour on a machine that
  has already completed it; verifying a permissions error as an admin; testing
  localisation with a forced locale header; "cannot reproduce" on a bug that only
  affects logged-out visitors; scraping or link-checking that trips bot
  protection and gets read as an outage; verifying an email render in a client
  that has already cached the assets.

### LL-0075 — An absolutely-positioned pseudo-element on a scroll container is part of what scrolls

- **Root Cause**: A horizontally-scrolling data table was given a right-edge
  gradient to advertise the scroll, written as `after:absolute after:right-0` on
  the same element that carried `overflow-x-auto`. An absolutely-positioned child
  of a scroll container is positioned against that container's *padding box*,
  which is the scrolled coordinate space — so the fade sat correctly only at
  `scrollLeft === 0`. The moment a user scrolled, it detached from the right edge
  and drifted left as a stray band floating over the data it was meant to frame.
- **Why It Happened**: The element that scrolls and the element that decorates
  were collapsed into one because they *look* like one thing in the markup, and
  the mistake is invisible in the state everyone tests: the unscrolled one. Every
  screenshot, every default render, and every jsdom test showed it correct. It
  is only wrong after an interaction that no unit test performs and no static
  capture reaches.
- **Solution**: Separate the two responsibilities. An outer, non-scrolling
  wrapper is `relative` and owns the `after:` fade; the inner element keeps
  `overflow-x-auto` and — critically — keeps `role`, `aria-label`, `tabIndex`
  and the focus ring, because **the focusable thing must be the scrollable
  thing**. Moving the a11y attributes up to the wrapper would trade one defect
  for a worse one: a focus ring on an element that does not scroll.
- **Preventive Rule**: **Never place an absolutely-positioned overlay on the
  element that owns `overflow`.** Put the overlay on a non-scrolling parent and
  let the scroller be a pure scroller. When reviewing any scroll affordance,
  ask what it looks like at `scrollLeft`/`scrollTop` *not* zero — and note the
  corollary defect: an unconditional fade also lies on wide screens, advertising
  a scroll that does not exist, because CSS alone cannot ask "does this
  overflow?" without JS measurement or scroll-state queries.
- **Similar Situations**: Sticky table headers inside an `overflow` wrapper;
  "scroll for more" chevrons; drop-shadows on carousels; a floating clear button
  inside a scrollable input; sticky column freezing in data grids; any
  `position: absolute` badge on a scrollable list; the same bug vertically, where
  a bottom fade rides up over content as the user scrolls down.

---

### LL-0076 — A runtime upgrade can ship a built-in global that shadows your test environment's implementation

- **Root Cause**: A frontend test suite began failing 22/22 in one file with
  `Cannot read properties of undefined (reading 'clear')` on `localStorage`,
  while CI stayed green. Node 26 had introduced an **experimental built-in
  `localStorage`** that requires a `--localstorage-file` flag to function.
  Without the flag it exists as a global but is undefined, and its mere presence
  shadowed the one jsdom provides. CI was unaffected only because it pins an
  older Node.
- **Why It Happened**: The failure named the application's own API, so it read
  as "our test setup is broken" rather than "the runtime changed underneath us".
  Nothing in the project had changed. The environment had — and a version
  manager was absent, so the local runtime had silently drifted ahead of the
  pinned one. The decisive clue was a warning Node prints itself and which is
  easy to scroll past: *"localStorage is not available because
  --localstorage-file was not provided."* The real scope was also wider than the
  symptom suggested: it was not one test file's problem, it was that
  `window.localStorage` was undefined for *every* test, which would have
  silently blocked any future feature that persisted state.
- **Solution**: Define the missing implementation in the test setup only when the
  runtime has left one missing — `if (!window.localStorage) { …jsdom-backed
  stub… }` — mirroring an existing `matchMedia` stub in the same file. The guard
  is the whole design: it is a no-op on the pinned runtime, so it cannot mask a
  real implementation or make local and CI diverge in the other direction. Prove
  both directions by running the suite under *both* runtimes.
- **Preventive Rule**: **When a test-only failure names a platform API, suspect
  the runtime before the code — and read the runtime's own warnings.** Establish
  the failing baseline *before* changing anything, so you know which failures are
  yours. Any environment stub must be guarded on absence and verified as a no-op
  on the pinned version. And note that a needed runtime is usually reachable
  without installing it (`npx node@24`, `docker run node:24`), so "I don't have
  that version" is rarely a reason to skip verifying against the pinned one.
- **Similar Situations**: `fetch`, `structuredClone`, `crypto.randomUUID`,
  `AbortSignal.timeout` or `Array.prototype.toSorted` landing natively and
  shadowing a polyfill; a browser shipping a global that collides with a library
  export; a language runtime adding a keyword or builtin that shadows a
  user-defined name; a base Docker image bumping and changing a default; any
  "works in CI, fails locally" where CI pins a version and the developer does not.

### LL-0077 — A test whose red state depends on luck is not a gate, however behavioral its failure

- **Root Cause**: A plan's TDD failing-step proved a repeat-avoidance fix by
  answering one session of 10 questions, redrawing the same category, and
  asserting no overlap. Against ~150 candidates a stateless draw repeats only
  about one seat in fifteen, so the pre-fix run **passed four times in five**.
  Run once, as TDD scripts usually are, it would have been recorded as a red
  state that had in fact never discriminated.
- **Why It Happened**: The code under test is randomized, and the assertion was
  written against the *population* the feature will eventually stress rather
  than one where the difference is forced. Both LL-0028 and mutation testing
  assume a deterministic verdict: neuter the logic, watch the test fail. Neither
  says what to do when neutering the logic makes the test fail only *sometimes*
  — which reads exactly like flake and gets retried away.
- **Solution**: Shrank the population until the correct implementation has no
  freedom: the narrowest published slice held 13 items, so draining 8 leaves
  exactly 5 unseen and the assertion becomes `drawn == those 5`. That is
  deterministic under the fix and lands by chance once in 1,287 runs. Verified
  by running the red state **five** times (5/5 failed) and the neutered
  implementation **eight** times (8/8 failed) — not once each. A second, weaker
  test in the same file was rewritten the same way after review measured its
  lazy-pass rate at ~24%.
- **Preventive Rule**: **When the code under test is randomized, compute the
  probability that your assertion passes against the broken implementation
  before you trust a single red run — and run the red state several times.** If
  that probability is not negligible, do not add retries; change the fixture so
  the correct behaviour is forced, then state the residual odds in a comment so
  the next reader can judge the gate. Prefer set equality on a small pool over
  disjointness on a large one: shrinking the population is what converts a
  statistical tendency into a deterministic assertion.
- **Similar Situations**: Sampling, shuffling, load-balancing and cache-eviction
  code; "assert the shuffled order differs" on a 3-element list; retry and
  backoff tests that pass because the first attempt happened to succeed; A/B
  bucketing assertions; property-based tests run at a case count too low to hit
  the interesting branch; any assertion of the form "X did not appear" where X
  was unlikely to appear anyway.

### LL-0078 — An arithmetic argument that a user-facing problem exists is a hypothesis, and the query that settles it usually takes minutes

- **Root Cause**: A feature was designed, specced, planned, implemented,
  reviewed and merged to remove repeated practice questions, on the strength of
  a table showing that by a student's fifth session more than half the questions
  would be ones they had already answered. The repeat rate was finally
  *measured* after the pull request was open: **1.7%** overall, 1.1% for the
  heaviest student. The table's assumptions — 60-question single-category
  sessions — did not describe how anyone actually used the product, which was
  10-question draws across a pool of 1,735.
- **Why It Happened**: The arithmetic was correct, internally consistent, and
  persuasive, so nobody asked what it was conditioned on. The data needed to
  check it was one SQL query against a table already being read for other
  purposes. The model was written into the spec as a table of numbers, which
  made it *look* like a measurement to every later reader.
- **Solution**: Measured drawn-versus-distinct questions per user; recorded the
  real figures in the spec beside the model, labelled which is which, and stated
  plainly that the work is insurance against a future state rather than a fix
  for a present defect — so a flat metric afterwards is not read as failure. The
  work still shipped: it was cheap, correct, and the projected problem is real
  once usage grows. Only the *framing* was wrong.
- **Preventive Rule**: **Before building a fix for a quantitative user-facing
  problem, measure the problem.** State the model's assumptions out loud and
  check each against real usage — session size, filters, population — because a
  model whose inputs are wrong produces a confident number, not an obvious
  error. Never let modelled figures sit in a spec formatted like measured ones;
  label them, and put the measurement next to them when it arrives. And note the
  companion trap: **a change can move its own headline metric the wrong way on
  purpose** — deliberately resurfacing missed questions *raises* the repeat
  rate — so define the metric to separate the defect from the feature, or the
  improvement will report as a regression.
- **Similar Situations**: Performance work justified by big-O rather than a
  profile; a cache added for a hit rate nobody sampled; capacity planning from
  peak-times-headroom instead of observed traffic; "users will hit the rate
  limit" without checking the distribution; any funnel or retention projection
  built from a per-step rate that was assumed rather than queried.

### LL-0079 — Promoting an optional code path to the default promotes its latent bugs with it

- **Root Cause**: A helper that read a learner's answer history used two
  unpaginated selects. The backing API caps an unpaginated read at 1,000 rows
  and returns success with no indication that a tail was dropped. This had been
  acceptable for months because only an opt-in mode called it. The change under
  review made it the input to **every** draw — silently converting a dormant
  truncation into a live one that would reintroduce the exact defect the change
  existed to remove, and could resurface an item the learner had already
  mastered, because "latest verdict" would be computed over a partial history.
- **Why It Happened**: The diff did not touch that helper, so neither the
  implementation nor the per-task reviews looked at it — the change was in who
  *calls* it, which a line-oriented diff does not show. The codebase already had
  a documented pagination rule and a named constant for it a few lines away; the
  helper predated the rule and nothing re-examined it when its caller set grew.
- **Solution**: A whole-branch review (as opposed to per-task review) caught it
  by reasoning about the new call graph rather than the changed lines. Both
  reads were paginated with an explicit sort key so pages cannot overlap or
  skip, with tests that model the cap and fail against both an unpaginated and
  an unordered read. Fixed on the same branch, before the promotion shipped.
- **Preventive Rule**: **When a change widens who calls something — a flag
  becoming the default, an opt-in path becoming automatic, an internal helper
  becoming public — re-audit that callee's assumptions against its new blast
  radius, even though its source is untouched.** Ask what was merely tolerable
  at the old volume or old caller set. Make the review's unit the call graph,
  not the diff; this class is invisible to per-task review by construction.
- **Similar Situations**: A beta flag flipped on for everyone; a debug or admin
  endpoint exposed to end users; a nightly batch query moved onto a request
  path; a helper with an in-memory cache, a fixed timeout, an unbounded
  accumulator or an N+1 that only mattered at low call volume; a test double
  promoted to production use; a dependency's optional feature becoming its
  default on upgrade.


---

### LL-0080 — A background browser tab does not render visibility-gated content, and the symptom looks like a lazy-load bug

- **Root Cause**: A browser-automation run drove a Chrome tab that was never
  brought to the foreground. The target application gates hydration of its feed
  on `document.visibilityState`, so every item stayed an unhydrated placeholder.
  Eight rounds of scrolling with waits produced exactly one hydrated item, and
  the extracted content was empty or wrong.
- **Why It Happened**: The failure presents as a *content* problem — empty DOM
  nodes, "the feed won't lazy-load past the first item" — so the instinctive
  responses are to scroll more, wait longer, or blame the selector, and all
  three were tried. Nothing in the tooling surfaces the tab's visibility:
  navigation reported success, the page had a valid DOM, and
  `document.hasFocus()` returned **true** while `visibilityState` was
  `"hidden"`, so the one signal that was checked gave false reassurance. A
  prior note recorded the workaround at symptom level ("the pane must be
  visible") without the mechanism, so the diagnosis was re-derived from scratch
  rather than applied.
- **Solution**: Queried `document.visibilityState` directly. It returned
  `"hidden"`; the human fronted the tab; it became `"visible"` and the feed
  hydrated immediately with no additional scrolling. Total fix time once the
  right question was asked: one query.
- **Preventive Rule**: **Before debugging "the content won't load" in browser
  automation, assert the environment renders at all — check
  `document.visibilityState === "visible"`, and treat a hidden tab as a broken
  precondition rather than as slowness.** Generally: when driving a UI you
  cannot see, verify the preconditions that UI's own rendering depends on
  (visibility, viewport size, focus, reduced-motion or headless flags) before
  trusting any negative observation about its content — an unrendered UI
  produces confident, wrong evidence. And when recording a workaround in memory
  or a runbook, capture the *mechanism*, not just the symptom: a symptom-level
  rule cannot be checked with a query, so it gets re-derived at full cost.
- **Similar Situations**: IntersectionObserver-driven infinite scroll;
  `requestAnimationFrame` loops, CSS animations and video that only advance
  while visible; timers throttled in background tabs; polling that silently
  stops when a tab is backgrounded; headless runs where an element is
  "invisible" because the viewport has zero height; screenshot tests against a
  minimized or offscreen window; any automation asserting absence of content it
  never gave the page a chance to draw.

---

### LL-0081 — A client cannot infer a deletion from a row's absence; tombstone ids must travel explicitly

- **Root Cause**: A sync API soft-deleted rows (`deleted_at`) and filtered them
  out of the pull payload, with a source comment stating this made "the
  deletion propagate to other devices". The client's merge only added and
  updated rows *present* in the payload; it never removed local rows missing
  from it. A row deleted on the laptop stayed on the phone indefinitely, and
  the two devices silently disagreed about what existed.
- **Why It Happened**: The server half looked finished — the column existed,
  the intent was written down, and deletion worked perfectly on the device
  that issued it (local removal plus a queued tombstone). The bug is invisible
  on the machine you are testing on, because that machine is the one that
  already applied the change. Nobody tested with a second client.
- **Solution**: Return the tombstoned ids from the GET and apply them in the
  merge. Unpushed local edits win over a remote delete — the queue is the only
  copy that exists nowhere else — and re-pushing revives the row. Critically,
  the *obvious* fix (prune local rows missing from the payload) would have
  caused data loss: locally converted legacy rows are deliberately not queued
  for upload and are also absent, so pruning would have deleted them before
  their server twin arrived.
- **Preventive Rule**: In any sync protocol, deletion must be an explicit
  signal, never absence from a payload. Absence is ambiguous — it also means
  "not created yet", "filtered", "paginated out", or "created locally and not
  yet sent". Before writing "remove what is missing", enumerate every reason a
  row can legitimately be absent; if that list has more than one entry, you
  need an explicit tombstone. Test deletion with two clients, because one
  client can never observe this failure.
- **Similar Situations**: Offline-first and CRDT/last-write-wins syncs, cache
  invalidation, directory mirroring (`rsync --delete`), incremental ETL where a
  missing source row could mean "deleted" or merely "not in this batch", and
  any reconciliation loop that treats desired-state absence as intent to
  destroy.

---

### LL-0082 — An action creator named after a state field silently replaces it when both are spread into one object

- **Root Cause**: A React context value was assembled as `{...state,
  ...actions}`. One creator, `syncError`, shared a name with the `syncError`
  state field. Spread second, the function overwrote the string. Every
  consumer then read a function — always truthy — so the UI took the error
  branch on every render, showing a permanent "SYNC ISSUE" on every device
  while sync was in fact healthy.
- **Why It Happened**: Both names are the natural one: the field *is* the
  error, the creator *sets* the error. The collision produces no error, no
  warning, and no type failure in plain JS. It also looked correct from the
  dispatching side, which destructured the same name and got exactly the
  function it wanted. And the app still worked — only the indicator lied,
  which reads as a cosmetic quirk rather than a bug.
- **Solution**: Renamed to `setSyncError` and moved the creators next to
  `initialState` so the shared namespace is visible in one place. A test now
  asserts the two key sets are disjoint, and that the merged value exposes the
  error as a value rather than a function.
- **Preventive Rule**: If state and actions are merged into one object, they
  share a namespace — assert in a test that their key sets are disjoint, since
  nothing else will tell you. Prefer verb-prefixed creators (`setX`, `clearX`)
  or nest them (`{...state, actions}`). Note the failure shape: a truthy
  function standing in for a value fails *silently and always*, so the symptom
  is a constant wrong state rather than an intermittent one — a UI element
  that never changes deserves as much suspicion as one that flickers.
- **Similar Situations**: Redux `mapStateToProps` + `mapDispatchToProps` merged
  into one props object, Zustand/Pinia stores mixing state and actions, any
  `{...defaults, ...overrides}` config merge, and template rendering contexts
  built from several sources.

---

### LL-0083 — Counting rows where the unit is people: one duplicate record read as two bodies

- **Root Cause**: Coverage statistics counted absence *rows*. "OUT TODAY" was
  the length of a filtered array, so a person with two live absences covering
  the same day counted twice. A genuine duplicate record — the same leave
  saved twice — would have reported one person as two bodies out for four
  days. The same mistake was present in five functions: the headline count,
  per-department coverage, returning-this-week, departing-soon, and
  person-days-lost. The last summed days per absence, so two overlapping
  absences billed the same day twice and the total could exceed the number of
  days the window contains.
- **Why It Happened**: Rows are what you have; people are what you mean. The
  mapping is 1:1 in clean data, so the code is correct right up until the data
  is not — and nothing in `filtered.length` names the unit it is counting, so
  it reads as obviously right at every review.
- **Solution**: Dedupe to the intended unit at the one definition every
  consumer routes through, with an explicit, documented choice of survivor —
  earliest start for "who is out", latest end for "who returns" (someone is
  not back until the last chit ends), earliest start for "who leaves next" —
  each tie-broken by id so row order can never change a count. Person-days
  became the size of a set of distinct `(person, day)` pairs.
- **Preventive Rule**: When a number is reported in a unit — people out,
  customers affected, days lost — the code must reduce to that unit
  explicitly. `rows.length` is only correct if something *enforces* a 1:1
  row-to-unit mapping, and nothing usually does. Name the unit in the function
  or its docstring. And when you find one instance, grep for its siblings
  immediately: this is a habit of mind, not an isolated slip, so it is
  reliably present more than once in the same file.
- **Similar Situations**: Daily-active-users from event rows, "customers
  affected" from ticket counts, seats from licence records, unique visitors
  from requests, stock on hand from movements — every case where the honest
  query was `COUNT(DISTINCT …)` and the written one was `COUNT(*)`.

---

### LL-0084 — n8n stores a workflow's nodes in two tables; patching only `workflow_entity` may not change what runs

- **Root Cause**: n8n 2.34.4 keeps a workflow's nodes in **both**
  `workflow_entity.nodes` and `workflow_history.nodes`, and
  `workflow_entity.activeVersionId` is a foreign key into
  `workflow_history(versionId)`. A direct database edit that updates only
  `workflow_entity` can therefore leave the runtime loading the history copy —
  succeeding as a write while changing nothing that executes.
- **Why It Happened**: Older n8n had exactly one place to edit, and a script
  written against that shape still reports "1 row updated". The extra columns
  (`versionId`, `versionCounter`, `activeVersionId`) and tables
  (`workflow_history`, `workflow_published_version`) arrived with the
  publish/versioning feature, and nothing about the update failing to take
  effect would be visible until the next scheduled run produced old output.
- **Solution**: Patch both rows in one transaction, leave `versionId` and
  `activeVersionId` untouched so the foreign key stays valid, and verify by
  md5-ing the stored code back out and comparing it to the source file.
  (`workflow_published_version` existed but held no row for this workflow, so
  the publish path was not in use.)
- **Follow-up (same day)**: importing a NEW workflow via
  `n8n import:workflow` creates the `workflow_history` row but leaves
  `workflow_entity.activeVersionId` **NULL**. Setting `active=1` was not
  enough — the scheduler silently skipped it, the boot log activated every
  other workflow, and the row still read `active=1`, so the database and the
  running state disagreed with no error anywhere. Pointing `activeVersionId`
  at the existing history version fixed it. **Activation follows the pointer,
  not the `active` flag**, which makes `active=1` a claim rather than a fact.
- **Preventive Rule**: Before editing any application's database directly,
  look for a history/versions/published table and for a column that *points*
  at it; assume the runtime may read through the pointer rather than the base
  row. After any activation change, verify against the runtime's own boot log
  or API, never against the flag you just wrote. Verify by reading the value back and hashing it against the intended
  source — "UPDATE … 1 row" only proves you wrote somewhere. Stop the
  container first (LL-0025), and after restarting confirm the boot log
  re-activates everything it did before.
- **Similar Situations**: Any tool with a draft-versus-published split
  (Strapi, Directus, Sanity, Airflow's serialized DAG table), feature-flag
  services with versioned configs, and anything exposing an
  `activeVersionId` / `publishedRevisionId` pointer.

---

### LL-0085 — A deployed PWA serves the previous bundle after a deploy, so a working fix looks broken

- **Root Cause**: `vite-plugin-pwa` with `registerType: 'autoUpdate'`
  precaches the app shell. After a deploy, the first page load installs and
  activates the new service worker, but the document keeps the assets it
  already has; only the *next* load runs the new bundle. Verifying a
  just-deployed fix on the first reload therefore tests the old code.
- **Why It Happened**: Every other signal said the deploy had landed — build
  green, deployment READY, the API returning new fields, the page reporting
  "Synced". `skipWaiting` and `clientsClaim` make the update feel immediate,
  but they govern *worker* activation, not the assets the current document
  already loaded. It cost two separate false conclusions that a correct fix
  had failed, and nearly a wasted debugging session.
- **Solution**: Compare the loaded bundle filename against the build output
  before judging behavior at all; force it by unregistering the worker and
  clearing `caches` (leaving `localStorage`, which holds the offline queue),
  or simply reload twice.
- **Preventive Rule**: When verifying a deployed change in a service-worker
  app, identify the running build *first* — hashed asset name or an explicit
  version string — and only then judge behavior. Treat "the fix did not work"
  as unproven until the running bundle is confirmed to contain it. Same family
  as LL-0011 and LL-0026 (wrong build masquerading as missing code), different
  mechanism: a precached shell rather than a stale server or wrong checkout.
- **Similar Situations**: Any service-worker app, CDN edge caching with a long
  `max-age`, mobile OTA bundles (Expo Updates, CodePush), and the browser
  bfcache serving a page that never re-executed.

---

---

### LL-0086 — macOS's bash 3.2 cannot parse `case` inside `$(...)`: it prints an error, exits 0, and assigns garbage

- **Root Cause**: `/bin/bash` on macOS is 3.2.57 (frozen at the last GPLv2
  release). A command substitution containing a `case`/`esac` statement —
  `x="$(case $v in a*) echo yes ;; *) echo no ;; esac)"` — is a parse error
  in that version. The damage is that it is a *non-fatal* one: bash writes
  the error to stderr, the assignment still completes with an empty or
  partial value, and the script's exit status stays 0.
- **Why It Happened**: The shape is idiomatic in bash 4+, so it reads as
  correct to anyone who learned shell on Linux or with Homebrew's bash first
  on `PATH`. It was written into an assertion that checked whether a
  destructive script had stopped a container. `bash -n` passes it clean — a
  syntax check does not exercise the substitution — so the usual pre-flight
  gate says nothing. The assertion silently compared against an empty string
  and reported success.
- **Solution**: Use a form 3.2 parses: `grep -qF`, a `[ ... ]` test, or
  parameter expansion. Where a `case` is genuinely wanted, run it as a
  statement and assign inside the branches rather than substituting the whole
  block.
- **Preventive Rule**: On macOS, treat `/bin/bash` as bash 3.2 and never
  assume 4.x syntax, regardless of what `bash --version` on `PATH` reports —
  scripts invoked as `bash foo.sh` or via a `#!/usr/bin/env bash` shebang may
  resolve differently than your interactive shell. Do not rely on `bash -n`
  to catch version-specific parse failures. When a shell assertion produces a
  surprising empty value, check stderr for a parse error before debugging the
  logic. Same family as LL-0035 (a check that cannot fail is not a check),
  different mechanism: here the interpreter, not the logic, silently voided it.
- **Similar Situations**: Any `#!/bin/sh` or `#!/bin/bash` script intended to
  run on both macOS and Linux; CI that runs on Linux while developers are on
  macOS (or the reverse); `sed`/`date`/`stat` BSD-vs-GNU flag differences,
  which fail loudly and are therefore *easier* than this one.

---

### LL-0087 — A self-healing safety net repaired the damage before the assertion looked, so the test could not fail

- **Root Cause**: A destructive upgrade script arms an `EXIT` trap that
  restarts the service if the script dies while the service is down. A drill
  asserted an ordering property — that a version lookup happens *before* the
  container is stopped — by checking, after a forced failure, that the
  container was still running. But if the ordering broke, the script stopped
  the container, died, and the trap restarted it and confirmed it healthy
  *before the process exited*. By the time the drill inspected, the container
  was running. The assertion passed either way.
- **Why It Happened**: The property under test and the recovery mechanism
  observe the same variable — "is it running now" — so the recovery is
  indistinguishable from the property holding. The safety net was added for
  good reasons and is genuinely correct; it just also erases the evidence.
  Mutation testing found it: relocating the lookup below the stop left the
  drill green, 4 assertions passing on a mutated script.
- **Solution**: Assert on evidence the recovery cannot manufacture. Two were
  added: the run's output must not contain the "stopping <container>" log
  line, and `docker inspect -f '{{.State.StartedAt}}'` must be unchanged
  across the run. The second is stronger — it catches a stop/restart that
  logs nothing.
- **Preventive Rule**: When testing a system that repairs itself, never
  assert on end state alone; the repair runs between the fault and your
  observation. Assert on a trace the repair does not rewrite — a log line, a
  monotonic timestamp, a call count, a generation counter. Ask explicitly:
  "if this property broke, what would be *different* by the time I look?" If
  the honest answer is "nothing", the assertion is decorative. Extends
  `docs/standards/testing.md` rule 7's discriminate-or-it-is-not-a-gate idea
  to systems with automatic recovery.
- **Similar Situations**: Supervisors and restart policies (`systemd`,
  Kubernetes liveness probes, `restart: unless-stopped`); retrying HTTP
  clients that mask a first-attempt failure; transactions that roll back
  cleanly so a constraint violation leaves no trace; any `finally`/`defer`
  cleanup that restores the state a test then reads.

---

### LL-0088 — A failed read that degrades to an empty result lets verification certify data loss

- **Root Cause**: A helper read a database's collection inventory and returned
  it as JSON, for use as a before/after baseline around a destructive
  upgrade: capture, upgrade, re-read, compare, roll back on mismatch. Three
  separate paths could return an *empty* inventory when the read had actually
  failed — an HTTP 200 with a malformed body, a per-item detail fetch that
  timed out and defaulted its count to zero, and an empty response body, which
  `jq` consumes with exit 0 and no output. Empty-compared-to-empty is a match,
  so the upgrade concludes "nothing changed" and reports success over data
  that may be gone.
- **Why It Happened**: Each degradation was individually reasonable — a
  `|| echo '{}'` here, a `2>/dev/null` there, a `// 0` default — and each was
  written to make the *happy* path robust. The failure only appears when you
  ask what the value is used *for*. All three were caught by review, in three
  separate waves, in code that already carried a comment explaining why the
  helper must never do this. Comments do not enforce.
- **Solution**: The helper prints nothing and returns 1 whenever the read did
  not genuinely succeed, distinguishing "the payload says zero items"
  (`jq -e '(.result.collections|type)=="array"'`) from "there was no usable
  payload". Callers abort rather than substituting a default, and the caller
  that had a separate status probe was fixed too: the probe and the read are
  different HTTP calls, so a 200 on the first does not license trusting the
  second.
- **Preventive Rule**: For any value that will be *compared* rather than
  merely displayed, make "could not read" a distinct outcome from "read
  successfully, and it is empty". Never let a default, a fallback, or a
  suppressed error stand in for data you did not obtain. The test for whether
  this matters: if the caller compares two reads and acts on equality, an
  unreadable value that looks empty will match another unreadable value and
  the action taken will be the dangerous one.
- **Similar Situations**: `curl | jq` pipelines generally; ORM `.first()`
  returning `None` for both "no row" and "query failed"; cache reads that
  return a miss on a connection error; `git diff` returning empty because the
  path was wrong; migration and backup verification of every kind.

---

### LL-0089 — Python's `HTTPServer` calls `socket.getfqdn()`, which hangs in a sandbox, so mock-based tests passed without ever running

- **Root Cause**: `http.server.HTTPServer.server_bind()` calls
  `socket.getfqdn()`, which performs a reverse-DNS lookup. In a
  network-sandboxed shell that lookup blocks for 5+ seconds. A test harness
  started the mock in the background and polled up to 5s for its port, timed
  out with an empty port, and then pointed the code under test at a malformed
  URL. Every request failed with connection-refused, which happened to satisfy
  the assertions — so three failure-mode tests passed while exercising none of
  the paths they named.
- **Why It Happened**: The harness reported green, and the assertions it
  contained were individually correct. Nothing distinguished "the mock served
  a malformed body and the code rejected it" from "there was no mock and the
  connection was refused". It was found only because a reviewer ran a mutation
  check: reverting the fix under test left the suite at 0 failures, proving
  the tests carried no signal.
- **Solution**: Set `socket.getfqdn = lambda *a, **k: "localhost"` before
  constructing `HTTPServer`. Additionally, assert that the mock actually came
  up — a nonempty port and one successful probe request — before the test
  proceeds, so a dead mock fails loudly rather than passing vacuously.
- **Preventive Rule**: A test whose setup can fail *soft* must assert its
  setup succeeded before asserting on behavior; otherwise the failure mode of
  the harness masquerades as the success of the code. Always mutation-check
  tests that depend on a fixture the test itself starts
  (`memory/reusable-patterns.md`, discriminating-test pattern). Prefer
  binding mocks to `127.0.0.1` with the FQDN lookup disabled in any sandboxed
  or offline environment.
- **Similar Situations**: `unittest.mock` patches applied to the wrong import
  path (patch succeeds, nothing intercepted); testcontainers or docker-compose
  fixtures whose readiness wait times out; WireMock/`nock`/`responses`
  interceptors that never match and fall through to a real connection error;
  any fixture started in the background and polled with a bounded wait.

---


### LL-0090 — A scope constraint that excludes the test directory ships the fix without the test that holds it

- **Root Cause**: A task brief limited edits to a fixed file list to protect a
  production-proven script during adjacent work. The fix — anchoring a pattern
  match to the end of a failure string — was proven against a scenario built in
  a **scratch copy** of the hermetic test harness, because the harness file was
  outside the allowed list. The scratch copy died with its temp directory, so
  the fix shipped with no committed test: reverting it to the old
  both-sides-wildcarded pattern left the entire suite green.
- **Why It Happened**: The scope constraint was correct and had a good reason.
  Its cost was invisible because the fix *was* proven — thoroughly, repeatedly,
  just not durably. At the moment of commit, "I verified this" and "this is
  protected by a committed test" felt like the same state, and nothing in the
  workflow distinguished them.
- **Solution**: Committed the reproducing scenario afterwards as its own commit,
  and mutation-tested it: restoring the pre-fix pattern fails exactly that
  scenario's four assertions and leaves the other nine green, proving the new
  scenario is the one carrying the signal.
- **Preventive Rule**: A scope constraint may freeze implementation files; it
  must never exclude the test that will hold the change. A brief whose file list
  contains no test file is a brief that is wrong, not a task that is small — say
  so before starting. Independently, before closing any fix: revert it locally
  and confirm the suite goes **red**. A fix whose removal keeps the suite green
  is untested, however carefully it was verified by hand.
- **Similar Situations**: any "touch only these files" brief; freeze windows
  around production-proven code (`memory/reusable-patterns.md`,
  freeze-and-unfreeze); subagent tasks handed an allowed-paths list; reviews
  that accept a demonstrated manual reproduction as evidence of coverage;
  hotfix branches where the test suite lives in a separate repo.

---

### LL-0091 — A helper that reads its caller's global by name cannot be shared, so the sibling with the same exposure keeps the bug

- **Root Cause**: An EXIT-trap safety net in `upgrade-n8n.sh` was hardened to
  confirm that its restart actually *held* — three settled state samples plus a
  `RestartCount` comparison across the window — in a function that read
  `$N8N_SAFETY_NET_SETTLE_SECS` directly. Its sibling `upgrade-qdrant.sh` had
  the identical exposure (same trap, same safety net, same
  `restart: unless-stopped` compose file) and went on confirming with a single
  container-state read taken the instant `docker compose up -d` returned. A
  container crash-looping on boot reads `Running=true` in exactly that instant,
  so the net printed `safety net: qdrant restarted` for a container that was
  dying — the single line an operator is most likely to trust without checking.
- **Why It Happened**: The function's *logic* was general from the first line;
  only its configuration reference was specific. That one variable name made it
  read as n8n-specific to everyone who looked, including the review that
  hardened it. The fix landed in one of the two places that needed it, and the
  gap was filed as a future task rather than recognised as a live defect in a
  shipped safety net.
- **Solution**: Moved the function into the shared library with the settle
  window as a **parameter**; each script passes its own `config.env` knob.
  Verified by mutation: collapsing the settle loop to a single sample, in the
  shared implementation, breaks the qdrant scenarios *and* the n8n scenarios
  from one edit — which is the evidence the two now share an implementation
  rather than owning two copies that will drift.
- **Preventive Rule**: When hardening a safety-critical path, grep for siblings
  with the same exposure **before** deciding where the fix lives — the question
  is "who else has this trap", not "where does this code currently sit". A
  shared helper takes its configuration as an argument; a helper that names a
  caller's global has silently chosen one caller, and will read as
  unshareable to the next person even when its logic is fully general.
- **Similar Situations**: paired upgrade/rollback or deploy/rollback scripts;
  per-service health and readiness checks; retry/backoff helpers reading a
  service-specific timeout; error-handling middleware pulling a global config
  object; any review note containing the phrase "the other one has the same
  problem".
- **Recurrence (2026-08-15, same repo, the rollback drills rather than the
  scripts)**: same shape, opposite direction. The **qdrant** drill had already
  been fixed to resolve its failed-upgrade directory by before/after set
  difference, carrying a comment explaining why "the newest one that exists" is
  the wrong question. The **n8n** drill it had been modelled on kept the
  original `ls | tail -1`, and went on reporting a proven post-migration
  rollback for runs that had set nothing aside. So the sibling that keeps the
  bug is not always the copy — here the fix landed in the copy and the ancestor
  was never revisited — and a written explanation sitting in the sibling file,
  which is the strongest form this warning can take short of shared code, was
  still not enough to propagate it. When a fix lands anywhere, the question
  stays "who else has this exposure", and the file it was copied *from* is on
  that list.

---

### LL-0092 — A shared bounded-wait helper's fixed poll interval is a latency floor every future caller pays

- **Root Cause**: A bounded-run helper backgrounds a command and polls
  `kill -0` once a second until a deadline. For `docker compose pull` that
  interval is free. A new caller — a restart confirm — made **five** bounded
  `docker inspect` calls per invocation, and because the command is
  backgrounded, the first poll essentially never finds it already finished
  (measured: 20/20 trials still running at the first check). So every call paid
  a full one-second floor.
- **Why It Happened**: `sleep 1` was correct and invisible for the helper's
  original callers, whose bounded commands took tens of seconds. The constant
  encoded an assumption about each call's *duration* that quietly stopped
  holding when a caller changed the *count*. Nothing in the helper stated the
  assumption, so there was nothing to notice.
- **Solution**: Probed the interval once at load — `sleep 0.2` where the shell
  accepts it, 1s otherwise, since fractional `sleep` is not POSIX — in the same
  shape as the existing `shasum`/`sha256sum` capability probe. Measured, all
  three numbers timed rather than inferred: the affected test file was 1m36s at
  100 assertions before the change, 2m45s at 117 assertions with the 1s poll,
  and 1m40s at 117 with the probe.
- **Preventive Rule**: A polling or backoff constant inside a shared helper is a
  latency floor on every future caller, not a local detail. State the assumption
  in the helper's comment where the constant is set, and re-measure whenever a
  new caller's per-invocation call count is an order of magnitude above the
  existing ones. Time it before and after — this cost was invisible in review
  and unmissable in `time`.
- **Similar Situations**: retry/backoff defaults, connection-pool acquire
  timeouts, debounce intervals in shared UI hooks, fixed `sleep` in CI wait
  loops, polling health-check clients, any helper whose cost profile was set by
  its first caller.

---

### LL-0093 — A condition dismissed as a test flake was the same condition silently failing in production

- **Root Cause**: A status check resolved "latest release" from GitHub's
  *unauthenticated* REST API (60/hr per IP). With the quota spent the call
  failed, the `available` version came back empty, `updateAvailable` computed
  to `false`, and **no finding was emitted** — so the unattended weekly job
  wrote a clean report for a stack it had never successfully asked about. The
  same 403 also failed one test assertion, and that assertion was the only
  visible symptom.
- **Why It Happened**: The test failure was labelled a **known flake** and
  documented with a workaround: "check the quota before believing the suite is
  red". A flake is by definition a thing you route around, so once it had a
  name and a workaround nobody asked what the same condition did on the
  production path — which shared the same helper and ran unattended every week.
  The documentation made it worse rather than better: it recorded that a
  rate-limited window produced "a false *could not detect* finding". That was
  wrong, and wrong in the comfortable direction — it produced no finding at
  all, which is why two sessions passed without anyone noticing.
- **Solution**: Authenticated the call with the token the machine's `gh` was
  already holding (60/hr → 5000/hr, nothing stored in the repo, unauthenticated
  fallback intact) so the condition is rare — and, the half that actually
  matters, made an unresolvable version an explicit finding so any future
  failure is loud rather than silent. The test assertion became a disjunction:
  a clean version **or** a reported failure, with the second branch exercised
  against a hermetic unreachable endpoint since it never runs while the real
  one is up. Per the new `docs/standards/security.md` rule 8, the token is
  passed through a `--config` file on stdin rather than `-H`, keeping it out of
  argv.
- **Preventive Rule**: Before writing "known flake" next to anything, trace the
  flaky condition through the **production** code path and state in the same
  note what it does there. A flake in a test that shares a helper with a
  scheduled job is a production failure that happens to have a witness.
  Separately: verify the effect you write down. A workaround note that asserts
  a benign symptom is the most durable hiding place a wrong belief can find,
  because every later reader treats it as already investigated.
- **Similar Situations**: any quota-bearing or rate-limited third-party API;
  tests sharing a client with a cron/scheduled job; "flaky in CI, fine locally"
  network assertions; retries that mask a persistent outage; anything where an
  empty result from a failed fetch flows into a boolean whose `false` means
  "nothing to do".

---

### LL-0094 — An invariant applied to one of two symmetric fields is not applied

- **Root Cause**: An amendment established "a failed detection must be
  reported, never silently treated as up to date", implemented as three
  hand-written checks over the `installed` field of three components. The
  sibling field `available` — same record, same consumer, same consequence when
  empty — got none of them, and neither did a fourth component added later. A
  failed availability lookup was therefore indistinguishable from "no update
  available", which is the exact outcome the amendment existed to prevent.
- **Why It Happened**: The invariant was written from the example that
  motivated it, a local tool that would not report its version. `installed` was
  the field in that bug; `available` had the same shape but a different failure
  mode (remote rather than local), so it never surfaced while thinking about
  the original case. Nothing enumerated the set the rule was meant to cover, so
  at review time "applied" and "applied to the case I was looking at" were
  indistinguishable — N-1 hand-written checks read exactly like N.
- **Solution**: Extended the check to `available` across all four components
  through a small helper, so adding a component is one call rather than
  remembering a rule, and covered it with a test that drives the real code path
  with the remote endpoint unreachable.
- **Preventive Rule**: When establishing an invariant over a **set** — fields
  in a record, routes in a router, variants in an enum — write the set down,
  and prefer a construct that iterates it over N hand-written checks. Where it
  must be hand-written, have the test assert over the enumerated set, so a new
  member fails loudly instead of inheriting silence. State the invariant in
  terms of the set ("every version field"), never in terms of the example that
  prompted it.
- **Similar Situations**: validation applied to some request fields; auth
  middleware on most routes; timeout/retry wrappers around most outbound calls;
  error handling covering the errors seen so far; sanitisation applied to the
  inputs someone remembered; any rule phrased "all X must Y" and implemented as
  a hand-maintained list.

---

### LL-0095 — A mutation-test that resets between the two operations under test cannot see a stale-state fault

- **Root Cause**: Two variables that a safety net reads together — a flag that
  decides whether it refuses to start anything, and the reason string it prints
  when it refuses — were paired into `arm`/`disarm` helpers so the reason could
  never describe a window that had already closed. The property "arming without
  a reason gets the generic default, never whatever the last window left
  behind" was tested as *arm-with-reason → disarm → arm-without-reason*. The
  mutant that implements exactly the fault being guarded against —
  `reason="${1:-$reason}"`, inheriting the previous window's text — **survived**
  the mutation pass. `disarm` had already reset the value to the default, so
  "inherit the current value" and "use the default" produced the same string.
- **Why It Happened**: The test was written to read as a realistic sequence
  rather than to isolate the property, and the operation sitting in the middle
  of that sequence was the *other half of the pair under test*. A restorer
  between two setters neutralises precisely the fault a stale-state test
  exists to catch, and the test still passes against the correct code, so
  nothing about the green run hints at it. It surfaced only because the mutant
  was run — and the first instinct on a surviving mutant is to interrogate the
  assertion, when the fault here was three lines above it, in the setup.
- **Solution**: Deleted the intervening `disarm` — *arm-with-reason →
  arm-without-reason*, back to back — which killed the mutant and, separately,
  matches the real shape of the code: the unknown-container-state abort arms
  the guard, attempts a compose revert, and re-arms with what it then knows.
  The realistic sequence and the discriminating one turned out to be the same
  sequence; the intervening reset was neither.
- **Preventive Rule**: When testing a pair where one operation restores what
  the other sets — arm/disarm, open/close, set/clear, acquire/release,
  begin/rollback, cache set/invalidate — at least one case must invoke the
  setter **twice with nothing in between**. Otherwise the restorer, not the
  code under test, is what makes the assertion pass. And when a mutant
  survives, suspect the test's *setup* before its assertion: an assertion can
  only observe what the arrangement left observable. Frameworks that reset
  state automatically between cases (`beforeEach`, fresh fixtures, auto-reset
  mocks) apply the same masking for free, to every test in the file.
- **Similar Situations**: any stateful pair as above; leak/carry-over tests
  that use one fixture per case; "the second call must not see the first
  call's data" written with a teardown in between; idempotency tests that
  reset between attempts; a `beforeEach` that re-seeds the exact field whose
  contamination is being tested.

---

### LL-0096 — A deferred defect's severity gets read off the note instead of re-derived, and grouping them assigns one severity to all

- **Root Cause**: Three follow-up items sat in a project README under a single
  bullet: "smaller, all recorded", with the first item's justification —
  provably unreachable code path, therefore a coherence gap and not a risk —
  standing in the reader's mind for all three. Two of them deserved that.
  The third did not: a rollback drill resolved the directory holding its
  evidence with `ls -d data.failed-* | sort | tail -1`, which answers "the
  newest one that exists" rather than "the one this run created". Residue from
  an earlier failed run satisfied the drill's post-migration proof for a run
  whose rollback had set nothing aside, and residue with a later-sorting stamp
  hid a genuinely broken rollback behind a passing assertion. Both cases
  exited **0** before the fix — a destructive-recovery drill reporting a proof
  it had never observed.
- **Why It Happened**: The three were found together, during one review, and
  were written down together because that is when they were noticed. Grouping
  by discovery time reads afterwards as grouping by severity. Every later
  reader — including the session that wrote the follow-up plan, and the one
  that eventually picked the work up — inherited the grouping and re-asked
  "is this worth doing?" once for the bullet rather than once per item. The
  bullet also stated its own conclusion ("coherence gap, not a risk"), which
  is the most efficient way to stop the next reader from checking.
- **Solution**: Fixed all three, and recorded the misfiling explicitly in the
  README's Known limitations rather than quietly marking the item done — so
  the next reader triaging a list inherits the correction rather than the
  original judgement.
- **Preventive Rule**: A deferred defect's severity is **re-derived when it is
  picked up**, never read off the note; the note records what was observed, not
  what it is worth. Never batch items of differing or unexamined consequence
  under one backlog line — one line per item, each stating what an operator or
  user would actually observe if it fired, in place of a verdict word. A
  written conclusion ("cosmetic", "unreachable", "harmless", "known issue")
  must carry the evidence for it inline, or it will be inherited instead of
  tested. Note especially that "unreachable" and "harmless" are different
  claims: the reachable one in this trio was the one that mattered.
- **Similar Situations**: "minor"/"cosmetic" backlog labels; TODO comments
  asserting their own severity; Known Issues sections; risk registers handed
  over between owners; batched code-review comments filed under one heading;
  any list where several findings share one justification sentence; a
  reviewer's "nit:" prefix on something that turns out not to be one.

---

### LL-0097 — A tool's silence is not evidence until the tool is confirmed to observe the thing

- **Root Cause**: A newly shipped browser telemetry feature appeared to send
  nothing on page exit. The investigation opened the in-app browser's network
  recorder, filtered for the endpoint, saw **no requests**, and concluded "the
  client never dispatched the request" — then spent the next several steps
  theorising about *why* dispatch would fail (a token awaited before `fetch`,
  a CORS preflight during unload). Both theories were wrong and the feature
  was never broken: that recorder does not capture **cross-origin** calls at
  all. It had only ever shown same-origin document and RSC requests. The API
  lives on a different host, so every single request the app made — including
  the ones that had demonstrably succeeded minutes earlier — was invisible to
  it.
- **Why It Happened**: A negative result from an instrument reads as strong
  evidence, because a positive result from the same instrument was trusted
  moments before. But "this tool shows me requests" was never established —
  only "this tool shows me *some* requests". The failure is asymmetric and
  that is what makes it dangerous: the tool had been right about everything it
  *did* show, so nothing about its output looked degraded. The absence was
  then reasoned *forward* from, generating plausible mechanisms for a
  phenomenon that did not exist.
- **Solution**: Switched to an instrument known to observe the thing — the
  server's own database rows and the platform's request logs, which had
  recorded every request all along. Three controlled reproductions then showed
  the exit path working, including a tab close with zero answers that
  delivered its complete timeline.
- **Preventive Rule**: Before treating "the tool shows nothing" as a finding,
  produce a **known-positive** through that same tool: trigger an event you
  are certain happens and confirm the tool reports it. If it cannot, the tool
  does not observe this class of event and its silence means nothing. Prefer
  the instrument closest to the effect being measured — the receiver's own
  records over the sender's observers. And state the instrument's scope out
  loud when reporting a negative: "no rows in the DB" and "nothing in this
  network pane" are not the same claim.
- **Similar Situations**: browser devtools filters that silently exclude a
  request type; log searches scoped to the wrong service, deployment or time
  window; `grep` over a directory the build actually reads from elsewhere;
  metrics dashboards whose retention or sampling drops the window under
  investigation; "no test failures" from a runner that collected no tests;
  "no such process" from a container that is not the one serving traffic.

---

### LL-0098 — A probe that mutates the state under test contaminates every reproduction after it

- **Root Cause**: An exit handler deduplicates itself with a closure flag —
  `hiddenReported`, set when a hide is reported and cleared only on the
  matching *visible* transition, so that one real exit firing two browser
  events cannot be counted twice. To test it without a real tab switch, the
  probe overrode `document.visibilityState` to `"hidden"` and dispatched a
  synthetic event. That worked, and correctly set the flag. Restoring the
  property afterwards fires **no** event, so the handler never ran its visible
  branch and the flag stayed set. The next reproduction — this time a genuine
  background transition — was therefore *correctly* deduplicated away, and
  produced exactly the symptom being hunted: no event, no request, nothing.
- **Why It Happened**: The probe was designed to observe the handler, and it
  did — but observing it meant driving it, and driving it advanced a state
  machine that outlives the probe. The contamination is invisible precisely
  because the code is behaving correctly: a dedup flag doing its job is
  indistinguishable, from the outside, from a handler that never fired. The
  second run also *looked* like independent confirmation of the first
  hypothesis, which is the worst possible outcome — a false positive that
  raises confidence.
- **Solution**: Reloaded the page for a fresh mount before the real
  reproduction, which cleared the closure. That run passed immediately, and a
  third — closing the tab outright — passed too, establishing the feature had
  never been broken.
- **Preventive Rule**: If a probe **drives** the code rather than only reading
  it, treat every later observation in that process as contaminated until the
  state is provably reset. Reload, remount, or restart between probe and
  reproduction. Where a property is faked to trigger a path, restoring the
  property is not enough — the *transition* the code listens for must also be
  replayed, or the code's view of the world stays where the probe left it.
  Prefer read-only observation (a passive listener, a log line) over
  synthesised events when the handler carries state across invocations.
- **Similar Situations**: dedup/idempotency flags, once-only guards, retry and
  circuit-breaker counters, rate limiters, feature-flag caches, and
  first-run/onboarding state; monkey-patched globals restored without
  re-firing their change events; a test double left installed for later
  assertions; manual DB edits made to force a code path, then not rolled back
  before the next attempt.

### LL-0099 — A raw NUL byte makes a file binary, and grep then reports no match for text that is plainly there

- **Root Cause**: Writing the JS/JSON escape for the NUL code point (backslash,
  `u`, four zeros) through an editing tool emitted an actual 0x00 byte instead
  of the six escape characters. One NUL anywhere in a file makes `grep` treat it
  as binary; with `-c` it printed nothing and exited 1, i.e. **"no match" for a
  string visible in the same file via `sed` and `tail`**. `file` reported the
  markdown as `data`, and `git diff` would have shown it as a binary blob.
- **Why It Happened**: The failure inverts the usual trust relationship. A grep
  that finds nothing reads as "verified absent", so a placeholder scan, a
  symbol-consistency check, and a secrets sweep all came back **clean because
  grep was matching nothing at all**. Nothing in the output says "I could not
  read this file"; the silence is indistinguishable from a pass. It recurred
  four times in one session, across a plan document, a source edit made by a
  subagent, a scratch ledger, and a report file — so it is a property of the
  tooling, not a one-off slip.
- **Solution**: Detect by byte count rather than by search:
  `python3 -c "print(open('F','rb').read().count(b'\x00'))"` must print `0`,
  and `file F` must not say `data`. Repair with a byte-level replace of `b"\x00"`
  by the literal escape text. `grep -a` will search a binary file as text and is
  the quickest way to confirm the file's *content* is otherwise fine.
- **Preventive Rule**: **Never accept a clean `grep` as verification of a file
  you just wrote.** Confirm the tool can see the file at all first — grep a
  string you know is present, or check `file`/byte counts — before trusting a
  zero-match result. More generally, when a check's pass condition is "no
  output", make it prove it examined the input.
- **Similar Situations**: any verification whose success is an empty result —
  secrets scans, lint-with-no-findings, `grep -L`, test filters that match zero
  tests and exit 0; also encoding faults that silently truncate a file at the
  first NUL, and CRLF/BOM issues that make anchored patterns fail to match.

### LL-0100 — An advisory that names the wrong subject reads as an all-clear for the right one

- **Root Cause**: A UI warning took `findings[0]` from a function returning
  findings across **all** departments, ordered by date then by a fixed
  department order. While logging leave for an ENT member it displayed a warning
  about *Orthopedics* — the earliest, first-ordered finding — and never showed
  the ENT collision the warning existed to catch.
- **Why It Happened**: The producer was correct and well tested; the defect was
  entirely in the **selection at the consumer**. `[0]` is a natural way to say
  "show one", and it is right whenever the list is already scoped to the subject
  — which it was during development, when the test fixture had only one
  department in play. The bug needs two departments in the window to appear, so
  every hand-check and the browser verification passed. Worse, the failure is
  not merely a miss: an amber line naming a *different* service line, rendered
  at the moment of approving this one, actively asserts that this line is fine.
  The design's stated rule was "never render an all-clear", and this violated it
  through the side door while appearing to honour it.
- **Solution**: Select by identity rather than by position —
  `findings.find(f => f.staffIds.includes(subjectId))` — so the warning speaks
  about the thing being edited, and renders nothing when the subject is in no
  finding.
- **Preventive Rule**: When a shared query returns results spanning several
  subjects and a consumer shows only one, **filter to the consumer's subject
  before choosing**, and never rely on ordering to do it. Ask what the warning
  asserts when it displays the *wrong* item: if a reader would take it as
  clearance for the item they are actually deciding, positional selection is a
  correctness bug, not a presentation choice.
- **Similar Situations**: `[0]`/`first()`/`LIMIT 1` over a multi-tenant or
  multi-entity result; showing "the next" appointment, alert, or error from a
  global list on a per-record screen; a single-slot notification fed by a
  many-subject queue; validation summaries that surface one message from a
  multi-field result.

---

<!--
Template for new entries — copy this block:

### LL-NNNN — Short, specific title

- **Root Cause**:
- **Why It Happened**:
- **Solution**:
- **Preventive Rule**:
- **Similar Situations**:
-->
