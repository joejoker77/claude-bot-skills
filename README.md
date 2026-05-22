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

## Authoring via bot

Bots can author skills here on behalf of their human users. Each bot has
a per-user PAT at `~/.skills-pat` (mode 600, fine-grained scope: contents
and pull-requests R/W on this repo only). The `skill-author` meta-skill
walks each bot through the workflow: source PAT → clone to `/tmp` →
branch → edit → push → open PR via the GitHub API.

Direct push to `main` is impossible — branch protection requires PR
plus the `scan` status check. Even if a bot is compromised and its PAT
leaks, the worst it can do is open a noisy PR. Humans gate merges.

When you (the human) want to ship a skill change:

- **Via your bot** — say "create a skill that …" or "edit the X skill to …".
  The bot drafts in `/tmp`, opens a PR, replies with the URL. You
  review on GitHub and merge.
- **Directly** — clone the repo locally with your own SSH/PAT, edit,
  push, PR, merge. Same gates apply.

Either way, after merge the change reaches affected bots within ~10
minutes via `skill-deploy@USER.timer`.

## Bootstrap state (2026-05-22)

- 2 skills: `proposal-generator` (restricted to dmrudenko via
  `users.yaml`), `skill-author` (deployed to all 8 bots).
- Deploy mechanism, CI, branch protection, per-bot author PATs all
  wired in subsequent commits.
