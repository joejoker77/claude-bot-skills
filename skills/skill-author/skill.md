# Skill: skill-author

Author or edit a skill in the `claude-bot-skills` GitHub repo by opening a
pull request. Do NOT edit files in `~/.claude/skills/` directly — that
directory is the rsync target of the deploy mechanism, and any local
changes are wiped on the next ~10-minute cycle.

## Trigger
"create a skill", "add a skill", "edit skill", "change the skill",
"скилл", "новый скилл", "измени скилл", "/skill-author"

## Setup (one-time check)

Before the first run, verify your env has the PAT:

```sh
test -r "$HOME/.skills-pat" && grep -q '^GITHUB_PAT_SKILLS=' "$HOME/.skills-pat" \
  && echo "ok" || echo "MISSING: ~/.skills-pat needs GITHUB_PAT_SKILLS"
```

If missing, ask the operator (Vitaliy) to provision it — do not try to
create it yourself.

## Steps

1. **Confirm with the user what they want.** Names, trigger phrases,
   expected behavior. If editing an existing skill, ask which one and
   what specifically should change.

2. **Source the PAT and set repo variables** (do this in a fresh subshell
   so the token doesn't leak into the parent shell environment):

   ```sh
   set -a
   . "$HOME/.skills-pat"
   set +a
   OWNER=joejoker77
   REPO=claude-bot-skills
   ```

3. **Clone the repo to a fresh temp dir** (do NOT reuse
   `~/.claude/skills-repo`, that one belongs to the deploy mechanism).
   Embed the PAT in the clone URL — git's `http.extraHeader` form
   alone does not satisfy the credential prompt, the URL form is what
   actually works:

   ```sh
   WORKDIR=$(mktemp -d /tmp/skill-author.XXXXXX)
   git clone --depth 1 \
       "https://x-access-token:$GITHUB_PAT_SKILLS@github.com/$OWNER/$REPO.git" \
       "$WORKDIR"
   cd "$WORKDIR"
   ```

4. **Create a branch** named for the change. Use kebab-case and a
   timestamp to avoid collisions:

   ```sh
   BRANCH="skill/$(date +%Y%m%d-%H%M%S)-<short-description>"
   git checkout -b "$BRANCH"
   ```

5. **Make the change**:
   - **New skill**: create `skills/<slug>/skill.md` with the standard
     skill format. See `skills/proposal-generator/skill.md` for a working
     example. Slug is lowercase kebab-case.
   - **Edit existing skill**: modify `skills/<slug>/skill.md` directly.
   - **Restrict to certain bot users**: append an entry to `users.yaml`
     with `users: [user1, user2]` and a `reason`. Skills NOT in
     `users.yaml` deploy to ALL 8 bots by default.

6. **Set commit author identity for the bot** (one-time per workdir):

   ```sh
   git config user.name "$USER-bot"
   git config user.email "${USER}+bot@ai-assistant.gg"
   ```

7. **Commit and push the branch** (same URL-with-PAT pattern as the
   clone; `http.extraHeader` alone doesn't work for push either).
   Do NOT pass `-u` — that would write the credentialed URL into
   `.git/config` as the upstream remote, leaking the PAT into the
   workdir. The clone + push + cleanup workflow is one-shot, no
   upstream tracking is needed:

   ```sh
   git add -A
   git commit -m "<short description of the change>"
   git push \
       "https://x-access-token:$GITHUB_PAT_SKILLS@github.com/$OWNER/$REPO.git" \
       "$BRANCH:$BRANCH"
   ```

8. **Open the PR via the GitHub API**:

   ```sh
   PR_BODY="Authored by the $USER bot at $(date -Is).\n\nUser asked: <quote>"
   curl -sS -X POST \
       -H "Authorization: Bearer $GITHUB_PAT_SKILLS" \
       -H "Accept: application/vnd.github+json" \
       "https://api.github.com/repos/$OWNER/$REPO/pulls" \
       -d "$(python3 -c 'import json,sys,os; print(json.dumps({
           "title": sys.argv[1],
           "head": sys.argv[2],
           "base": "main",
           "body": sys.argv[3],
           "maintainer_can_modify": True
       }))' "<PR title>" "$BRANCH" "$PR_BODY")" \
     | python3 -c 'import sys,json; d=json.load(sys.stdin); print(d.get("html_url") or d)'
   ```

9. **Clean up and report back to the user**:

   ```sh
   cd /
   rm -rf "$WORKDIR"
   ```

   Send the user the PR URL printed in step 8 via the Telegram reply
   tool. Tell them:
   - CI is running skill-scanner against the change
   - To merge, they review on GitHub and click "Squash and merge"
   - Within ~10 minutes after merge, the new skill will be on the
     appropriate bots (per `users.yaml` whitelist)

## Notes

- **Branch protection** is enabled on `main`. You CANNOT push directly to
  main. PR is the only path.
- **CI** runs `skill-scanner scan-all --use-llm` on every PR. If it finds
  HIGH/CRITICAL issues, the PR can't be merged until they're addressed.
  This is the safety check.
- **PAT scope** is `Contents: R/W` + `Pull requests: R/W` + `Issues: R/W`
  for this repo only. Cannot administer the repo or push to main. If you
  hit a permission error doing anything outside PR/commit, that's the
  scope — escalate to operator.
- **Never echo the PAT** to any log, output, or message. Use it inline in
  curl/git commands. The `http.extraHeader` form keeps it out of git's
  config files.
- **One PR per change** — don't bundle unrelated skill edits.
- **The skill name in the slug should be unique** across `skills/`. If a
  slug already exists and the user wants a fresh skill, pick a different
  slug (don't overwrite).
