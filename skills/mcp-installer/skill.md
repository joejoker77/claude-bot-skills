# Skill: mcp-installer

Install or remove an MCP server for THIS bot, using the
`claude-bot-skills` git repo as the source-of-truth for templates and
the OneCLI vault as the store for any API keys / tokens.

The user never has to know about git, PRs, vaults, or systemd — they
just say "поставь мне Slack MCP" and this skill drives the whole thing.

## Trigger

User intent: install / connect / hook up an MCP server.

"install MCP", "поставь MCP", "подключи slack mcp", "хочу github mcp",
"добавь mcp", "поставь mcp-сервер для X", "remove mcp",
"убери mcp X", "/mcp-installer".

DO NOT confuse with Composio integrations (OAuth services like Gmail,
Calendar, etc.). Composio is for OAuth-backed user-facing services and
uses `composio-connect`. MCP-installer is for low-level MCP servers
that need raw API tokens (filesystem, github raw API, postgres, redis,
slack-with-bot-token, etc.).

When in doubt: if the user says "Gmail / Calendar / Drive / Notion /
Slack-as-me" → Composio. If they say "give Claude direct access to my
postgres / github org / filesystem / S3 bucket" → MCP-installer.

## Pre-flight

This skill needs:

1. The `claude-bot-skills` repo is already synced into
   `~/.claude/skills-repo/` (skill-deploy.timer keeps it fresh every
   ~10 minutes — should already be there).
2. A scoped GitHub PAT at `~/.skills-pat` (only needed when proposing
   a new MCP template via PR; same one `skill-author` uses).
3. A narrow sudoers entry letting this bot call
   `/usr/local/sbin/mcp-set-secret vitaliy *` without password. The
   operator (Vitaliy) installs that once per bot. If it's missing,
   step 5 will fail with "sudo: a password is required" — report
   that to the user and ask the operator.

Run this preflight before doing anything else:

```sh
test -d "$HOME/.claude/skills-repo/mcp" && echo "REPO_OK" || echo "REPO_MISSING"
sudo -ln 2>&1 | grep -q "mcp-set-secret" && echo "SUDO_OK" || echo "SUDO_MISSING"
```

If `SUDO_MISSING`: tell the user the operator needs to add the
sudoers entry — do NOT try to add it yourself. Continue past steps
1–4 anyway (they don't need sudo), but stop before step 5.

## Steps

### 1. Identify what the user actually wants

Ask the user, plain language, what MCP they want and what it should
have access to. Map it to a slug.

- "Slack" → `slack`
- "GitHub" → `github`
- "Postgres" / "Postgresql" → `postgres`
- "Filesystem" / "файлы" → `filesystem` (already deployed for vitaliy)
- "Redis" → `redis`

If unclear, ask. Don't guess.

### 2. Check whether a template already exists

```sh
SLUG="<the slug>"
TPL="$HOME/.claude/skills-repo/mcp/$SLUG/template.json"
test -f "$TPL" && echo "TEMPLATE_EXISTS" || echo "TEMPLATE_MISSING"
```

- **TEMPLATE_EXISTS** → skip to step 4.
- **TEMPLATE_MISSING** → continue to step 3 to propose a new template.

### 3. (Only if template missing) Propose a new MCP template via PR

You are about to author a new MCP template in the repo. This is the
analogue of `skill-author` for MCPs.

a. **Confirm with the user**: explain you don't have a template for
   this MCP yet, you'll add one via PR, the maintainer (Dmitrii via
   CI) will review, and it'll auto-merge if it passes the scanners.
   Ask the user for permission to do that. They might just say "yes,
   go ahead".

b. **Determine the MCP server package**. For official MCPs, look up
   `@modelcontextprotocol/server-<slug>` on npm. For community MCPs,
   search github or ask the user for the npm package / git URL. If
   uncertain, run `mcp__exa__web_search_exa` with query
   `"MCP server" "<slug>" npm OR github`.

c. **Source PAT** (in a subshell to avoid leaking):

   ```sh
   set -a
   . "$HOME/.skills-pat"
   set +a
   OWNER=joejoker77
   REPO=claude-bot-skills
   ```

d. **Clone fresh, branch, author template.json + README.md** under
   `mcp/<slug>/`. Template schema (canonical):

   ```jsonc
   {
     "mcp_stanza": {
       "command": "npx",
       "args": [
         "-y",
         "@modelcontextprotocol/server-<slug>"
       ],
       "timeout": 30000
     },

     // Non-secret per-user values (set in users.yaml, kept in git):
     "user_config": {
       "workspace_id": {
         "description": "Slack workspace ID this bot will operate in",
         "type": "string",
         "required": true
       }
     },

     // Per-bot secrets — value lives in OneCLI vault, header injected on outbound:
     "secrets": {
       "bot_token": {
         "host_pattern": "slack.com",
         "header_name": "Authorization",
         "value_format": "Bearer {value}",
         "description": "Slack bot user OAuth token (xoxb-...)"
       }
     }
   }
   ```

   Notes on schema fields:
   - `host_pattern`: the API hostname OneCLI gateway watches for. Only
     outbound requests to this host get the header injected.
   - `header_name`: HTTP header the server reads (`Authorization`,
     `X-API-Key`, etc.).
   - `value_format`: how the raw value is wrapped before injection.
     Use `Bearer {value}` for OAuth Bearer tokens, `{value}` for raw.
   - Every entry under `secrets` becomes a required step in step 5.

   Write README.md alongside, modelled on
   `mcp/filesystem/README.md` (purpose, schema, per-user fields,
   security notes).

e. **Commit, push, open PR with auto-merge** — same recipe as
   `skill-author` steps 6–10 (see `skills/skill-author/skill.md`).
   Set PR title `Add MCP template: <slug>`. Body should include:
   - What the MCP does (one paragraph)
   - The npm/git source URL
   - What secrets it needs

f. **Tell the user** the PR is up, CI is running, on green it
   auto-merges, and within ~10 min after that the template will
   be available for the install step. Tell them to message you
   again to continue once they see the merge land (or wait
   ~15 min and re-trigger this skill).

   This is the natural pause point. Don't try to poll for merge in
   this skill — let the user resume.

### 4. Read template, list required secrets

```sh
TPL="$HOME/.claude/skills-repo/mcp/$SLUG/template.json"
SECRETS=$(jq -r '.secrets // {} | keys[]' "$TPL")
USER_CONFIG_KEYS=$(jq -r '.user_config // {} | keys[]' "$TPL")
```

Read the README for human-readable context on what each secret is and
where the user gets it from (e.g., "Slack: Settings → Apps → OAuth →
Bot User OAuth Token, starts with xoxb-").

If `USER_CONFIG_KEYS` is non-empty: ask the user for those values
(non-secrets — paths, IDs). You'll need to edit `users.yaml` to
record them — that's another PR (do it together with step 5 if
needed). For an MCP with no user_config (like a token-only Slack
install), skip this.

### 5. Collect secrets and bind them via mcp-set-secret

For each secret name `S` in `$SECRETS`:

a. **Ask the user via Telegram** for the value. Be explicit about
   the format (e.g., "Slack bot token, starts with `xoxb-`"). Tell
   them:
   - The value will be stored in OneCLI vault on the host
   - The bot process itself will never see the raw value — OneCLI
     gateway injects it into outbound HTTPS requests
   - It can be removed later with the same skill (--remove)

b. **Receive the value via Telegram message.** Once you have it,
   IMMEDIATELY pipe it into `mcp-set-secret` and DO NOT echo it
   anywhere else — not in your reply, not in any debug output, not
   in logs:

   ```sh
   # value is in shell variable $V (don't print it)
   printf '%s' "$V" | sudo -n /usr/local/sbin/mcp-set-secret \
       "$(id -un)" "$SLUG" "$S" 2>&1
   ```

   The `printf '%s'` (no trailing newline) is intentional —
   `mcp-set-secret` uses `read -rsp` which strips a trailing newline
   anyway, but using `printf` is more deterministic than `echo`.

   If exit code is non-zero, show the output (it shouldn't contain
   the value) and stop.

c. **Forget the value.** Unset the shell variable, do not keep it in
   your context. If the user asks you to "double-check" — refuse,
   tell them to test the MCP instead.

### 6. Update users.yaml to whitelist this bot

If this MCP is not already listed in `users.yaml`, you need to add an
entry so the next `deploy-mcp` cycle picks it up. This is a PR (CI
validates: rate-limit, sensitive-paths).

But if the MCP IS already in `users.yaml` and the bot is already in
the users list, skip — your bot will pick it up via the existing
entry.

The current users.yaml (pull latest first) might already list the
MCP with `users: [other_bot]`. In that case:

a. Edit `users.yaml` to add your bot to the users list of that MCP:

   ```yaml
   mcp:
     slack:
       users:
         - other_bot
         - vitaliy        # added
       user_config:
         vitaliy:
           workspace_id: T0123456
   ```

b. Same PR mechanics as `skill-author`. Title: `Enable MCP <slug>
   for vitaliy`.

c. After merge, within ~10 min, deploy-mcp will reconcile and the
   MCP appears in `~/.claude/settings.json`.

### 7. Verify it landed

```sh
sleep 15
jq '.mcpServers."'"$SLUG"'" // "not deployed"' \
    ~/.claude/settings.json
```

If still `"not deployed"` — `deploy-mcp` hasn't run yet (it runs
once per ~10 min via `mcp-deploy@vitaliy.timer`). The user can
either wait or, if they have operator access, the operator can
trigger it manually:

```
sudo systemctl start mcp-deploy@vitaliy.service
```

Don't try to do this from the bot — it's not in the sudoers
whitelist.

### 8. Tell the user

Plain language:
- "Готово, MCP `<slug>` подключён. Перезагружу Claude чтобы он его
  подхватил."
- Request a graceful self-restart so the bot picks up the new
  `settings.json` on next session. Operator runs:
  `graceful-restart-bot vitaliy`
- Or: bot can suggest it but NOT trigger it.

After restart, the new MCP tools become callable as
`mcp__<slug>__*`.

## Removing an MCP

If the user wants to remove an MCP:

```sh
sudo -n /usr/local/sbin/mcp-set-secret "$(id -un)" "$SLUG" \
    "$SECRET_NAME" --remove
```

(once per secret). Then PR removing the bot from `users.yaml` for
that MCP (or removing the whole entry if no bots want it anymore).

## Security & operational notes

- **Never log, echo, or paste secret values back to the user.** Once
  the value is piped to `mcp-set-secret`, scrub the variable.
- **Never write the raw value to a file.** Not in `/tmp/`, not in
  the repo, not in logs.
- **The `mcp-set-secret` script reads from stdin via `read -rsp`** —
  piping `printf '%s' "$V" | sudo mcp-set-secret …` is the correct
  shape. Verified by inspecting the script.
- **One PR per template** — don't bundle multiple MCP additions in
  one PR (rate-limit-gate and sensitive-paths-gate will be confused).
- **CI runs on every PR**:
  - `signature-scanner` (skill-scanner + mcp-scanner) catches
    YARA/LLM-flagged content.
  - `sensitive-paths-gate` blocks PRs touching `.github/`, `deploy/`,
    `users.yaml` without a human label.
  - `rate-limit-gate` blocks >5 PR/author/24h.
- **If CI fails**, the PR will not auto-merge. Read the output, fix
  the template, push again. Don't fight the CI — its job is to
  catch you.
- **Don't paper over user_config validation errors.** If `deploy-mcp`
  warns that a required user_config field is missing for your bot,
  that's a real problem — add the value to `users.yaml`, don't
  skip.

## Related skills

- `skill-author` — for non-MCP code skills (the analogue of this
  skill, for skills/ instead of mcp/).
- `proposal-generator` — example of a real per-bot-restricted skill.
