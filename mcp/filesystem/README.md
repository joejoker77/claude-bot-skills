# MCP: filesystem

Official `@modelcontextprotocol/server-filesystem` — gives Claude read/write
access to ONE directory on disk (the path is set per-user, not in this repo).

## Schema (template.json)

This template uses the v2 schema:
- `mcp_stanza` — the actual MCP server config that lands in settings.json
- `user_config` — per-bot non-secret values (file paths, IDs, etc.)
- `secrets` — (this MCP has none) per-bot HTTP credentials stored in OneCLI vault

## Per-user config

Set in `users.yaml`:

```yaml
mcp:
  filesystem:
    users: [vitaliy]                          # which bots get this MCP
    user_config:
      vitaliy:
        fs_root: /home/vitaliy/work/mcp-sandbox
    reason: "..."
```

| Key       | What it is                                  | Example                       |
|-----------|---------------------------------------------|-------------------------------|
| `fs_root` | Absolute path to the directory MCP can see  | `/home/<user>/work/sandbox`   |

`fs_root` is a path, NOT a credential — it's safe to live in git per user.

If `fs_root` is missing for a user listed in `users`, deploy-mcp **skips this MCP for that user**.

## Notes

- Pointing `fs_root` at a sensitive directory grants Claude full read/write
  there. Pick the path deliberately.
- For MCPs that DO need API keys (Slack, GitHub, etc.), they go in OneCLI
  vault via `mcp-set-secret <user> <slug> <secret-name>` — bot processes
  never see the raw value.
