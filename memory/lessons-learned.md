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

<!--
Template for new entries — copy this block:

### LL-NNNN — Short, specific title

- **Root Cause**:
- **Why It Happened**:
- **Solution**:
- **Preventive Rule**:
- **Similar Situations**:
-->
