# claude-bot-skills

Source of truth for custom skills installed on the Claude Telegram bot fleet
(host `185.168.208.235`, 8 bots: artemii, boris, daria, dmrudenko, leonid,
team, vitaliy, viveanne).

Plugin-provided skills (e.g. `telegram:access`, `telegram:configure` from the
`claude-plugins-official/telegram` plugin) are **not** managed here — they
ship with their parent plugin. This repo only owns hand-authored skills
under each bot's `~/.claude/skills/`.

## Layout

```
.
├── skills/                       # one directory per skill, name = skill slug
│   └── proposal-generator/
│       └── skill.md              # the skill body
├── users.yaml                    # per-skill bot whitelist
├── .github/workflows/            # CI: skill-scanner on every PR
├── CONTRIBUTING.md               # rules for adding/editing a skill
└── README.md
```

## How a skill gets deployed

1. Author opens a PR with the new/changed skill under `skills/<slug>/`.
2. CI runs the Cisco **skill-scanner** (`scan-all --recursive`) against
   `skills/`. PR fails if any HIGH/CRITICAL finding appears. LOW/MEDIUM
   findings post as PR annotations but don't block.
3. Reviewer approves, PR merges to `main`.
4. Each bot runs a systemd timer (`skill-deploy@<user>.timer`, every 10 min)
   that:
   - `git pull` this repo into `/var/lib/claude-bot-skills/repo/`
   - reads `users.yaml`, computes which skills are allowed for this user
   - `rsync -a --delete` allowed skills into `/home/<user>/.claude/skills/`
   - logs to `/var/log/skill-deploy/<user>.log`
5. The bot's own `skill-install-gate` (PreToolUse hook, Phase D) does **not**
   interfere — deploy is done by root via rsync, not by the bot itself
   writing files.

## Security gates summary

| Gate                                       | When             | Owner            |
|--------------------------------------------|------------------|------------------|
| CI: skill-scanner on PR                    | author opens PR  | GitHub Actions   |
| Code review on PR                          | author opens PR  | repo maintainers |
| `skill-install-gate` (PreToolUse hook)     | bot tries `Write`/`Edit` under `~/.claude/skills/` | each bot |
| `skill-scanner-gate@<user>.timer`          | every 15 min     | systemd on host  |

## Per-user customisation

A skill is deployed to **all 8 bots by default**. To restrict, add an entry
in `users.yaml` listing the allowed bot users — see that file's header for
the schema.

## Bootstrap state (2026-05-22)

- 1 skill: `proposal-generator` (migrated from dmrudenko's bot, restricted
  to dmrudenko via `users.yaml`).
- Deploy mechanism and CI wired in subsequent commits.
