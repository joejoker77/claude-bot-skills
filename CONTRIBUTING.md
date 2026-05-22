# Contributing skills to this repo

## Adding a new skill

1. Pick a kebab-case slug: e.g. `weekly-digest-poster`.
2. Create `skills/<slug>/skill.md`.
3. Use the standard skill format (see `skills/proposal-generator/skill.md`
   for a working example):
   - `# Skill: <slug>` heading
   - One-line description of what it does
   - `## Trigger` — keywords/phrases that should activate the skill
   - `## Steps` — numbered, concrete actions
   - `## Notes` — gotchas, dependencies, secrets needed
4. If the skill depends on external resources (binaries, secrets, mounts)
   that don't exist on every bot, add it to `users.yaml` with the
   restricted user list and a one-line `reason`.
5. Open a PR. CI runs `skill-scanner scan-all --recursive` against
   `skills/` — fix any HIGH/CRITICAL findings before requesting review.

## Skill body checklist

- [ ] Trigger keywords are unambiguous (won't fire on unrelated chat)
- [ ] Steps reference absolute paths or `~/`-relative paths, never CWD
- [ ] Any required env vars / secrets are listed under `## Notes`
- [ ] No hardcoded API keys, tokens, or passwords in the skill body
- [ ] If skill exec's a script, that script is checked into source control
      somewhere reachable from the target bot (not just on one machine)

## Editing an existing skill

- Bump the meaning, not the wording, only when necessary. Triggers that
  fire on customer-facing keywords ("invoice", "оффер", etc) are touched
  with care — silent broadening can fire the skill on chats it shouldn't.
- If you change the user whitelist in `users.yaml`, mention it in the PR
  description. After merge, on the next deploy cycle (~10 min) the skill
  will appear/disappear from affected bots.

## What CI checks

`.github/workflows/skill-scanner.yml` runs on every PR touching `skills/`:

- `skill-scanner scan-all --recursive --format json skills/`
- Fails if any finding is HIGH or CRITICAL severity
- Posts LOW/MEDIUM findings as PR annotations (not blocking)
- Optionally runs the LLM-as-judge analyzer when an OpenRouter key is
  available in repo secrets (`OPENROUTER_API_KEY`).

## Local sanity check

Before opening a PR, run the scanner locally:

```sh
skill-scanner scan-all --recursive --format json skills/ | jq '.findings'
```

The scanner is installed on `185.168.208.235` at `/usr/local/bin/skill-scanner`.
