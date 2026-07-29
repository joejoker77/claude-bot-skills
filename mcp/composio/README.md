# MCP: composio

Composio tool-router access (Gmail, Drive, Calendar, Slack, Notion, GitHub,
Stripe, Shopify, Figma, and 500+ other apps) for the bot, on behalf of the
user's connected accounts.

## Schema (v2)

- `mcp_stanza` — lands in `~/.claude/settings.json` → `mcpServers.composio`.
  It's a **stdio** server: `/usr/local/bin/composio-mcp-bridge`.
- `user_config` — **none**. Routing is per-user but resolved server-side, not by us.
- `secrets` — **none** in the vault. The bridge holds no credentials.

## Why the bridge (and not a per-user HTTP tool_router URL)

`composio-mcp-bridge` is a tiny stdio↔unix-socket shovel to the root-side
Composio proxy daemon (`/run/composio-proxy/sock`). The daemon:

- identifies the calling bot by **linux user** via `SO_PEERCRED` (kernel-provided,
  unforgeable from userspace), and
- holds the Composio org `api_key` + the per-user `tool_router` (`trs`) map
  root-side in `/etc/composio/proxy.toml` — never in the bot's pod.

This makes the template **uniform across all bots**: no per-user `${USER_CONFIG}`
`trs` id in git, no `${SECRET}` in the vault. Every bot gets the identical stanza;
per-user routing + auth happen root-side. (This supersedes the earlier
vitaliy-pilot form that put a per-user `https://backend.composio.dev/tool_router/trs_.../mcp`
URL directly in `.mcp.json` — that required a per-bot `trs` and wasn't codifiable.)

## Setup (per bot)

Nothing to bind. The bridge deploys to every bot by default (no secret gate).
Prerequisite handled by the operator: the bot's `trs` entry exists in the
root-side `/etc/composio/proxy.toml` map (created when the user first connects
an app via the Composio connect flow).

## Removing

Restrict in `users.yaml` under `mcp.composio.users` to the allowed bots, or
drop the bot's entry from the root-side proxy map.

## Notes

- Per-bot isolation is enforced by the kernel (`SO_PEERCRED`): bot A cannot make
  Composio calls routed as bot B — the daemon keys off the connecting linux uid.
- The bridge is our code (`/usr/local/bin/composio-mcp-bridge`), not an npm
  package; it is bind-mounted read-only into every pod.
