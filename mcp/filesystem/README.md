# MCP: filesystem

Official `@modelcontextprotocol/server-filesystem` — gives Claude read/write
access to ONE directory on disk (the path is set per-user, not in this repo).

## Required per-user secrets

Set via `mcp-set-secret <user> filesystem`:

| Key       | What it is                                  | Example                       |
|-----------|---------------------------------------------|-------------------------------|
| `FS_ROOT` | Absolute path to the directory MCP can see  | `/home/<user>/work/sandbox`   |

If `FS_ROOT` is not set for a user, deploy-mcp **skips this MCP for that user** —
it never accidentally points at another user's path.

## Notes

- The path lives in each bot's own `/etc/mcp-secrets/<user>/filesystem.env`,
  not in this repo and not in Telegram chats.
- Pointing FS_ROOT at a sensitive directory grants Claude full read/write
  there. Pick the path deliberately.
- This MCP is allowed only for bots listed in `users.yaml` under
  `mcp.filesystem.users`. If the field is absent, all bots get it (same
  convention as skills).
