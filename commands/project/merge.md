Merge a pull request with the appropriate strategy based on the source branch.

**Input:** $ARGUMENTS
- If it's a PR number (e.g., `42` or `#42`), use that PR.
- If it's a PR URL, extract the PR number from it.
- If empty, detect the PR associated with the current branch: `gh pr view --json number,headRefName,baseRefName`

---

## Protected Branches
These branches must NEVER be committed to directly: `main`, `master`, `uat`, `prod`, `production`, `preprod`, `preproduction`, `staging`, `tests`, `integration`, `qa`, `develop`.

---

## Step 1: Identify the PR

1. Resolve the PR number from `$ARGUMENTS` or the current branch (see Input above).
2. Fetch PR metadata:
   ```bash
   gh pr view <number> --json number,title,headRefName,baseRefName,state,body,commits
   ```
3. If the PR is not open → **STOP**: "PR #<number> is not open (state: <state>). Aborting."
4. Extract `headRefName` (source branch) and `baseRefName` (target branch).

---

## Step 2: Choose merge strategy

### Worktree branches → Squash merge
If the source branch matches `feat/*`, `clean/*`, or `spec/*`:
- Strategy: **squash merge**
- Build a concise squash commit message:
  1. Read the PR title and body
  2. Read the commit list: `gh pr view <number> --json commits --jq '.commits[].messageHeadline'`
  3. Compose the merge message:
     - **Subject line**: the PR title (e.g., `feat: add OAuth support`)
     - **Body**: a short summary (3-5 bullet points max) of what was done, synthesized from the commits — not a raw dump of commit messages

### Protected/integration branches → Merge commit
If the source branch is one of the protected branches listed above (e.g., merging `develop` into `main`, `staging` into `prod`):
- Strategy: **merge commit** (preserves full commit history)
- Use the default merge commit message

### Other branches → Ask the user
If the source branch doesn't match any pattern above, use **AskUserQuestion** to ask:
- "Branch `<headRefName>` doesn't match a known pattern (feat/*, clean/*, spec/*, or protected). Which merge strategy should I use?"
- Options: "Squash merge", "Merge commit"

---

## Step 3: Check PR status

Before merging, verify the PR is mergeable:
```bash
gh pr checks <number>
```
- If checks are failing → **STOP**: "PR #<number> has failing checks. Fix them before merging."
- If checks are pending → ask the user whether to wait or force merge.
- If no checks or all passing → proceed.

Also check for merge conflicts:
```bash
gh pr view <number> --json mergeable --jq '.mergeable'
```
- If `CONFLICTING` → **STOP**: "PR #<number> has merge conflicts. Resolve them before merging."

---

## Step 4: Merge

### Squash merge
```bash
gh pr merge <number> --squash --subject "<subject>" --body "<body>"
```

### Merge commit
```bash
gh pr merge <number> --merge
```

---

## Step 5: Cleanup

After a successful merge, **always** perform all cleanup steps (unless `--delete-branch=false` is passed):

1. **Delete the remote branch immediately:**
   ```bash
   git push origin --delete <headRefName>
   ```
   - If the remote branch was already deleted (e.g., by GitHub auto-delete), ignore the error and continue.

2. **Switch back to the base branch:**
   ```bash
   git checkout <baseRefName>
   git pull origin <baseRefName>
   ```

3. **Clean up worktree** (if the source branch was a worktree branch `feat/*`, `clean/*`, `spec/*`):
   - Check if a worktree exists: `git worktree list | grep "<headRefName>"`
   - If found → remove the worktree: `git worktree remove <worktree-path>`
   - Prune worktree references: `git worktree prune`

4. **Delete the local branch:**
   ```bash
   git branch -D <headRefName>
   ```

5. **Confirm to the user:**
   - PR merged successfully
   - Merge strategy used
   - Remote branch deleted
   - Worktree cleaned up (if applicable)
   - Local branch deleted
   - Current branch is now `<baseRefName>` and up to date


---

## Optional arguments

If `$ARGUMENTS` contains additional flags:
- `--delete-branch=false` → skip branch deletion (local, remote) and worktree cleanup
- `--force` → skip check status verification
