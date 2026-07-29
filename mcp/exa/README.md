# MCP: exa

[Exa](https://exa.ai) web search + crawling for the bot: `web_search_exa`,
`web_search_advanced_exa`, `get_code_context_exa`, `crawling_exa`,
`company_research_exa`, `people_search_exa`, `deep_researcher_start`,
`deep_researcher_check`. This is the fleet's canonical web-search path
(built-in `WebSearch`/`WebFetch` are denied).

## Schema (v2)

- `mcp_stanza` — lands in `~/.claude/settings.json` → `mcpServers.exa`.
  An **http** server at `https://mcp.exa.ai/mcp`. The URL is **keyless** —
  the API key is NOT in the URL (unlike the legacy `?exaApiKey=<key>` form,
  which leaked the key into every config file on disk).
- `user_config` — none.
- `secrets.exa-api` — the Exa API key. Lives in the OneCLI vault, bound to
  `<bot>-bot`. The gateway proxy injects it as `x-api-key: <value>` on
  outbound HTTPS to `mcp.exa.ai`; the bot process never holds it.

## Setup (per bot)

```sh
sudo /usr/local/sbin/mcp-set-secret <bot-user> exa exa-api
# paste the shared Exa key; value never echoed
```

Or, since the key is shared and already in the vault, bind the existing
secret object to the bot's agent:

```sh
onecli agents set-secrets --id <bot-agent-uuid> --secret-ids <...,exa-api-id>
```

After binding, `deploy-mcp` writes `mcpServers.exa` on the next cycle.

## ⚠️ Review note — deploy gating

The stanza is keyless and relies on **host-pattern injection** at the proxy
(matches the validated vitaliy-pilot config). Because there is no
`${SECRET:exa-api}` marker *inside* the stanza, `deploy-mcp` will not
auto-skip exa on bots that lack the binding — it deploys the stanza to all
bots, and exa simply fails at runtime on any bot without `exa-api` bound.

If you prefer clean skip-when-unbound behavior (like github), change the
stanza to carry the marker explicitly:

```json
"headers": { "x-api-key": "${SECRET:exa-api}" }
```

Left keyless here to reproduce the exact working reference; flag for the
reviewer to pick.

## Notes

- Shared key across the fleet is fine; per-bot vault binding preserves
  isolation (bot B can't use bot A's key even if compromised).
- Legacy inline-key form (`.mcp.json` with `?exaApiKey=...`) should be
  removed from each bot once this stanza lands, to kill the on-disk leak.
