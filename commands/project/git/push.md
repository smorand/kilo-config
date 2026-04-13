Push the current branch and create a pull request.

## Protected Branches
These branches must NEVER be pushed to directly: `main`, `master`, `uat`, `prod`, `production`, `preprod`, `preproduction`, `staging`, `tests`, `integration`, `qa`, `develop`.

## Pre-flight

1. Run `git branch --show-current` to identify the current branch.
2. If on a protected branch → **STOP immediately**: "You are on `<branch>`, a protected branch. You must be on a working branch (spec/*, feat/*, clean/*) to push. Aborting."
3. Run `git status` — if there are uncommitted changes, ask the user whether to commit them first or abort.

## Determine the target branch

1. Run `git remote show origin | grep 'HEAD branch'` to find the default remote branch (usually `main` or `master`).
2. Use that as the PR target branch.

## Push

1. Push the current branch to remote: `git push -u origin <current-branch>`
2. If push fails, report the error and stop.

## Create or Update Pull Request

1. Check if a PR already exists for this branch:
   ```bash
   gh pr view --json number,url,state 2>/dev/null
   ```

### If a PR already exists (and is open)
1. Update the PR body with the latest commit log:
   - `git log <target-branch>..HEAD --oneline` to get all commits
   - Format as a summary with bullet points
2. Update the PR:
   ```bash
   gh pr edit <number> --body "<updated body>"
   ```
3. Return the PR URL and tell the user: "PR #<number> updated with latest changes."

### If no PR exists
1. Build the PR title from the branch name:
   - `spec/add-oauth` → "spec: add OAuth"
   - `feat/health-check` → "feat: health check"
   - `clean/compliance` → "clean: compliance"
2. Build the PR body from the commit log since diverging from the target branch:
   - `git log <target-branch>..HEAD --oneline` to get all commits
   - Format as a summary with bullet points
3. Create the PR:

```bash
gh pr create --title "<title>" --body "$(cat <<'EOF'
## Summary
<bullet points from commit log>

## Test plan
- [ ] All tests pass
- [ ] Documentation is up to date
EOF
)"
```

4. Return the PR URL to the user.

## Optional arguments

If `$ARGUMENTS` contains additional context (e.g., a reviewer username, a label), incorporate it:
- `--reviewer <username>` if a reviewer is mentioned
- `--label <label>` if a label is mentioned
- `--draft` if the user says "draft" or "WIP"
