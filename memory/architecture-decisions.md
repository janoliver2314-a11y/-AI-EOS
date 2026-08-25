# Architecture Decisions Index

Fast-lookup index of Architecture Decision Records. Full reasoning lives in
`docs/decisions/`. This file is a summary; if it disagrees with an ADR, the
ADR wins — update this index to match.

| ID | Title | Status | Summary |
|---|---|---|---|
| 0001 | Record Architecture Decisions | Accepted | Use ADRs under `docs/decisions/` for significant, hard-to-reverse decisions. See `docs/decisions/0001-record-architecture-decisions.md`. |
| 0002 | Always-on services on EliteDesk via SSH tunnel | Accepted | n8n + qdrant run 24/7 on the Ubuntu home server; Mac keeps localhost URLs through a LaunchAgent SSH tunnel, avoiding config/OAuth churn. See `docs/decisions/0002-services-on-elitedesk-via-ssh-tunnel.md`. |

<!--
Add one row per ADR, most recent first is not required — keep in ID order.
Status is one of: Proposed, Accepted, Superseded, Deprecated.
When an ADR is superseded, update its Status here and link forward/back in
the ADR files themselves.
-->
