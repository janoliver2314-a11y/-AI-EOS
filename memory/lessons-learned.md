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

<!--
Template for new entries — copy this block:

### LL-NNNN — Short, specific title

- **Root Cause**:
- **Why It Happened**:
- **Solution**:
- **Preventive Rule**:
- **Similar Situations**:
-->
