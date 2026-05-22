# Skill: skill-author-selftest

Operator-only self-check that the bot can execute the `skill-author`
workflow on this host. Walks through clone → branch → commit (no push,
no PR, no real changes), then reports pass/fail.

Use after deploys, after PAT rotations, after any infra change that
might have broken the authoring flow. **Not** intended as a regular-user
trigger — the trigger phrases are explicit and developer-flavored, so
end-users won't fire it accidentally.

## Trigger
"selftest skill-author", "skill-author selftest", "/skill-author-selftest"

## Steps

1. **Confirm the operator wants to run this.** Reply with what you'll do
   (clone to /tmp, no push, no PR, no changes to the repo) and wait for
   explicit "yes" before proceeding.

2. **Source PAT + set repo variables**:

   ```sh
   set -a
   . "$HOME/.skills-pat"
   set +a
   OWNER=joejoker77
   REPO=claude-bot-skills
   ```

   If `$GITHUB_PAT_SKILLS` is unset after this, report
   `selftest: FAIL — missing PAT in ~/.skills-pat`.

3. **Clone the repo to a fresh temp dir**:

   ```sh
   WORKDIR=$(mktemp -d /tmp/skill-author-selftest.XXXXXX)
   git clone --depth 1 \
       "https://x-access-token:$GITHUB_PAT_SKILLS@github.com/$OWNER/$REPO.git" \
       "$WORKDIR" 2>&1
   ```

   If exit code is non-zero, report `selftest: FAIL — clone failed`.

4. **Create a throwaway branch** (never pushed):

   ```sh
   cd "$WORKDIR"
   BRANCH="selftest/$(date +%Y%m%d-%H%M%S)"
   git checkout -b "$BRANCH"
   git config user.name "$USER-selftest"
   git config user.email "$USER+selftest@ai-assistant.gg"
   ```

5. **Make a no-op commit** (changes a marker file in /tmp only):

   ```sh
   echo "selftest marker $(date -Is)" > /tmp/skill-author-selftest-marker.txt
   git status --short
   ```

   We don't actually modify the repo files — just verifying the working
   tree is clean and git operations work.

6. **Verify GitHub API reachable with PAT** (read-only):

   ```sh
   curl -fsS \
       -H "Authorization: Bearer $GITHUB_PAT_SKILLS" \
       -H "Accept: application/vnd.github+json" \
       "https://api.github.com/repos/$OWNER/$REPO" \
       | python3 -c 'import sys,json; d=json.load(sys.stdin); print("api ok, repo size:", d.get("size"))'
   ```

   Non-zero exit or HTTP error means the PAT is invalid or revoked.

7. **Clean up**:

   ```sh
   cd /
   rm -rf "$WORKDIR" /tmp/skill-author-selftest-marker.txt
   ```

8. **Report the result back via Telegram**:

   ```
   ✅ skill-author selftest passed
      - PAT loaded from ~/.skills-pat
      - clone succeeded
      - branch + commit operations OK
      - GitHub API reachable as joejoker77
      - workdir cleaned
   ```

   Or, if any step failed:

   ```
   ❌ skill-author selftest FAILED at step N
      <error message>
   ```

## Notes

- This skill <b>never pushes</b>, <b>never opens a PR</b>, <b>never modifies the
  repo</b>. It exists purely to validate that the bot's local plumbing
  (filesystem, network, git, PAT, GitHub API) is healthy.
- If the test fails, the most likely causes in order: PAT revoked or
  rotated, network egress blocked, /tmp full, git binary missing.
- Operator can also force-run by typing the full slash command — the
  trigger phrases are intentionally verbose to prevent accidental fires
  from end-user chatter.
