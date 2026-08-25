# ADR 0002: Always-On Services Run on the EliteDesk Server, Reached via SSH Tunnel

- **Status**: Accepted
- **Date**: 2026-08-25

## Context

The n8n automation stack (LNC outreach auto-logging, follow-up reminders,
dashboard API) and the shared qdrant vector store ran in Docker Desktop on
the MacBook M5. Workflows therefore only executed while the laptop was
awake — schedule triggers and Gmail-label polling silently stopped every
time the lid closed. An HP EliteDesk (Ubuntu 26.04 LTS, x86_64, 14 GB RAM)
was set up as a dedicated always-on host.

A large amount of configuration assumes `http://localhost:5678` (n8n) and
`http://127.0.0.1:6333` (qdrant) on the Mac: the LNC dashboard code, the
`lnc-outreach-tracker` skill, the n8n MCP registration, Claude Code
permission allowlists, and — most fragile — the Google OAuth authorized
redirect URIs.

## Decision

1. Migrate the `lnc-n8n` and `qdrant` compose stacks to the EliteDesk
   (`~/stacks/<name>`), keeping images pinned to the same versions and
   containers bound to `127.0.0.1` on the server.
2. Keep every consumer pointed at `localhost` on the Mac, satisfied by a
   persistent SSH local-forward tunnel (LaunchAgent
   `com.janoliver.elitedesk-tunnel`, forwarding 5678/6333/6334,
   `KeepAlive` + `ExitOnForwardFailure`).
3. Expose nothing on the LAN except SSH (UFW default-deny, allow 22).
4. Local-dev containers (Supabase `supabase start` scaffolding) stay on
   the Mac; only genuinely always-on services move to the server.

## Alternatives Considered

- **LAN IP / hostname for the services**: cleaner topology, reachable from
  any device, but requires touching ~8 config/doc locations, adding a new
  Google OAuth redirect URI, and exposing unauthenticated n8n webhooks to
  the LAN. Rejected: high blast radius for no functional gain today.
- **Tailscale**: best for off-LAN access, but adds a dependency and still
  requires the URL/OAuth churn of the LAN-IP option. Deferred — can layer
  on later without undoing this decision.
- **`docker save`/`load` image transfer**: invalid — Mac images are arm64,
  server is amd64. Fresh pulls of the same pinned multi-arch tags instead;
  only bind-mounted data directories were copied (tar over SSH, checksums
  verified, WAL+db+shm moved together after a clean `compose down`).

## Consequences

- Workflows and schedules now run 24/7 independent of the laptop.
- The tunnel occupying Mac ports 5678/6333 doubles as a guard: the retired
  Mac copies of the stacks cannot be accidentally `compose up`'d.
- Server ops require SSH (`ssh elitedesk`); `sudo` there prompts for a
  password by design (the setup-time NOPASSWD grant was revoked).
- Ubuntu 26.04 ships sudo-rs; its stricter, case-sensitive `visudo` is a
  lesson recorded in `memory/lessons-learned.md`.
- If the Mac is off, nothing breaks server-side; the tunnel (and thus
  Mac-local access) simply resumes when the LaunchAgent reconnects.
- **Pending:** the `ops/stack-updates` upgrade harness still targets the
  local Docker daemon and the retired Mac stack copies. Its upgrade
  scripts are guarded against accidental use and its weekly check is
  unloaded until the harness is ported to run on the server (planned as
  its own session).
