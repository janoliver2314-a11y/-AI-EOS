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

<!--
Template for new entries — copy this block:

### LL-NNNN — Short, specific title

- **Root Cause**:
- **Why It Happened**:
- **Solution**:
- **Preventive Rule**:
- **Similar Situations**:
-->
