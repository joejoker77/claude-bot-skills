# MCP: github

Official [`@modelcontextprotocol/server-github`][upstream] from
modelcontextprotocol/servers. Lets the bot interact with the GitHub API
on behalf of a user: read repos, list issues / PRs, comment, search code,
manage releases — exact capabilities depend on the scopes of the token
you provide.

[upstream]: https://github.com/modelcontextprotocol/servers/tree/main/src/github

## Schema (v2)

- `mcp_stanza` — the stanza that lands in `~/.claude/settings.json` →
  `mcpServers.github`.
- `user_config` — none (path is provided at the server-side, not by us).
- `secrets.github_pat` — the PAT that authorizes the MCP's API calls.
  Lives in the OneCLI vault, bound to `<bot>-bot` agent. The raw value
  is **never** stored in `settings.json` or any file on disk; OneCLI's
  gateway proxy injects it on outbound HTTPS requests to `api.github.com`
  as `Authorization: Bearer <value>`.

## Setup (per bot)

Use the `mcp-installer` skill or run manually:

```sh
sudo /usr/local/sbin/mcp-set-secret <bot-user> github github_pat
# interactive: paste PAT, value never echoed
```

After: `deploy-mcp` is auto-triggered and `mcpServers.github` appears in
`~/.claude/settings.json` on the next ~10-min mcp-deploy cycle, or
immediately when `mcp-set-secret` calls it directly.

## Token scopes

Pick the **least permissive** scope the user actually needs.

| Use case                          | Classic scopes              |
|-----------------------------------|-----------------------------|
| Read-only public repo access      | `public_repo` (or nothing)  |
| Read private repos                | `repo:status`, `read:org`   |
| Open / comment on PRs and issues  | `repo`, `read:org`          |
| Push commits / manage releases    | `repo`, `workflow`          |

Fine-grained PATs are preferred when the user's org allows them — they
let you restrict to specific repos.

## Removing

```sh
sudo /usr/local/sbin/mcp-set-secret <bot-user> github github_pat --remove
```

This deletes the secret from the vault, unbinds it from the bot's
agent, and re-runs `deploy-mcp` so the stanza disappears from
`settings.json` (since the required secret is no longer bound).

## Notes

- Per-bot isolation: bot A's PAT is bound only to `A-bot` agent. Bot B
  cannot make API calls to `api.github.com` using A's token even if
  compromised — the proxy literally won't inject A's secret on B's
  requests.
- Rate limits: GitHub's API limit is per-PAT, 5000 req/h for classic
  PATs on personal accounts. Multiple bots sharing the same PAT (NOT
  recommended) would share that limit.
- The MCP itself is upstream code; we don't fork. New `@modelcontextprotocol/server-github`
  releases pick up automatically on the next `npx` resolution (no pin).
