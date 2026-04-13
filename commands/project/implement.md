Implement changes in the current project, either from a specification file, a backlog entry, or a direct description.

**Input:** $ARGUMENTS

**Default mode: FULL** — This command runs thorough, end-to-end implementation by default: code → tests → deploy → verify. The user can opt out of specific phases (e.g., "skip deployment", "no tests") but the default is the complete pipeline.

---

## Workflow Overview

| Phase | Name | Purpose |
|-------|------|---------|
| 1 | Pre-flight | Git state, branch, input parsing, resume detection, context loading |
| 2 | Skill Detection | Load tech-specific skills, discover Makefile targets |
| 3 | Implementation | Plan, deps, code loop with checkpoint commits, quality gates, error recovery |
| 4 | Verification & Testing | Unit/integration/E2E tests, non-regression, full test run |
| 5 | Deployment Verification | Deploy, functional testing via browser/API, non-regression post-deploy, human-blocked items |
| 6 | Documentation & Finalization | Docs, sync with main, archive spec, summary |

---

## PHASE 1: PRE-FLIGHT CHECKS

### 1.1 Git clean state & git setup

#### Protected Branches
These branches must NEVER be committed to directly: `main`, `master`, `uat`, `prod`, `production`, `preprod`, `preproduction`, `staging`, `tests`, `integration`, `qa`, `develop`.

#### Pre-flight
1. Run `git status` — if the working tree is dirty (uncommitted changes), **STOP immediately** and tell the user to commit or stash before proceeding.
2. If git is not initialized, **STOP immediately** and tell the user to first initialize git. Don't initialize yourself.
2. Run `git branch --show-current` to identify the current branch.

#### Context validation & branch setup
- If on a `feat/` branch → continue (iterating on implementation). Verify remote sync:
  - `git fetch origin && git rev-list --count HEAD..origin/<current-branch>` (ignore errors if remote doesn't exist)
  -n If count > 0 → **STOP**: "Branch is behind remote. Resolve manually."
- If on a `spec/` or `clean/` branch → **STOP**: "You are in a `{branch}` branch which is not meant for implementation. Switch to the main project directory to create a feat branch."
- If on a protected branch or any other branch → create a new feat/ branch:
  1. Derive a slug from the spec filename or description (e.g., `add-oauth-support`, `health-check-endpoint`)
  2. Ensure to work on this branch

### 1.2 Parse input

Determine the input type and handle accordingly:

- **Spec file path** (e.g., `specs/2025-01-15_add-oauth.md`): Validate the file exists, read the full spec. No plan mode — the spec is the plan.
- **Backlog reference** (`from backlog`, `BL-NNN`, or keywords):
  1. Read `specs/BACKLOG.md`
  2. If `BL-NNN` ID is given → find that exact entry
  3. If keywords are given → search for matching entries
  4. Handle matches:
     - **0 matches** → STOP: "No matching backlog entry found for '<query>'. Check `specs/BACKLOG.md`."
     - **1 match** → use it
     - **N matches** → present the matches and ask the user to pick one using AskUserQuestion
  5. Assess complexity of the selected entry:
     - **Simple** (single concern, clear scope, no architectural decisions) → proceed directly with implementation
     - **Complex** (multi-module, architectural decisions, unclear scope) → suggest the user run `/project:spec` first to produce a full spec. If the user chooses to proceed anyway, use plan mode.
  6. After successful implementation, mark the backlog entry as done (strikethrough or move to a "Done" section)
- **Text description**: Assess complexity:
  - **Simple** → proceed directly
  - **Complex** → enter plan mode or suggest `/project:spec`

### 1.3 Read context
- Read `CLAUDE.md` and relevant `.agent_docs/*.md` for project conventions
- Check `tests/` directory structure and existing test patterns
- Note what's already covered so you don't break it

### 1.4 Resume detection

When on an existing `feat/` branch (not freshly created):
1. Check for prior commits: `git log main..HEAD --oneline`
2. If commits exist:
   - Read the spec or description associated with this branch (from commit messages or the original spec file)
   - Cross-reference completed work against spec requirements
   - Present a summary: "Found N prior commits on this branch. Already done: [list]. Remaining: [list]."
   - Continue implementation from where things left off — do NOT redo completed work

### 1.5 Understand requirements

Before writing any code:
- Read the spec/description carefully, cross-reference with existing code, identify ambiguities
- If questions remain about requirements, scope, or approach, **ask them upfront** using AskUserQuestion before writing any code. Don't guess — clarify.

---

## PHASE 2: SKILL DETECTION & CONTEXT

### 2.1 Load tech-specific skills

Detect the project's tech stack and load the appropriate skills using the **Skill** tool. Check these indicators:

| Indicator | Skill to load | Notes |
|-----------|---------------|-------|
| `go.mod` or `*.go` files | `golang` | |
| `pyproject.toml` or `*.py` files | `python` | |
| `iac/` folder with `*.tf` files | `terraform` | |
| `pubspec.yaml` | `flutter` | If available |
| `package.json` with React/Vue/Svelte | — | `frontend-design` is a plugin that activates automatically, not a skill to load |

Load **all applicable skills** — a project can have multiple (e.g., Python + Terraform).

**Precedence rule:** When a loaded skill's conventions conflict with the project's `CLAUDE.md`, ask the user what to do and the level of inconsistency discovered.

If no tech-specific skill matches, proceed without loading one but follow general best practices from CLAUDE.md.
If skills match, ensure to follow all the rules and guidelines provided inside the skill.

### 2.2 Discover project commands

Parse the project's Makefile (or equivalent) to identify available quality targets. Store these as `QUALITY_COMMANDS`:

```
QUALITY_COMMANDS:
  format:    <command or "not available">   (e.g., make format)
  lint:      <command or "not available">   (e.g., make lint)
  build:     <command or "not available">   (e.g., make build)
  typecheck: <command or "not available">   (e.g., make typecheck, npx tsc --noEmit)
  test:      <command or "not available">   (e.g., make test)
  check:     <command or "not available">   (e.g., make check — often runs all of the above)
  deploy:    <command or "not available">   (e.g., make deploy)
```

If no Makefile exists, use direct tool commands (e.g., `go fmt ./...`, `ruff format .`, `pytest`). Document what you'll use.

---

## PHASE 3: IMPLEMENTATION

### 3.1 Plan the work

Before writing any code, create a task list (using TaskCreate) with all the work items derived from the spec or description. This gives the user visibility into progress.

For each task, include:
- What will be implemented
- Which files will be affected
- Acceptance criteria (from spec requirement identifiers if available)

### 3.2 Install dependencies

Before coding, check if the spec or description introduces new dependencies:
1. Identify new packages/modules required
2. Install them using the project's package manager:
   - Python: `uv add <package>`
   - Go: `go get <package>`
   - Node: `npm install <package>` (or `yarn add`, `pnpm add` based on lockfile)
3. Run dependency sync if needed (e.g., `uv sync`, `go mod tidy`)

### 3.3 Implementation loop

For each task in the task list:
1. **Mark in_progress** (TaskUpdate)
2. **Implement** the change:
   - Follow the loaded skill's conventions
   - Follow existing project patterns found in CLAUDE.md and .agent_docs/
   - If implementing from a spec, follow the requirements (using whatever ID scheme the spec uses) systematically
   - Keep changes focused — one logical unit at a time
3. **Run quality gates** in order (skip any that are "not available"):
   1. Format (`QUALITY_COMMANDS.format`)
   2. Lint (`QUALITY_COMMANDS.lint`)
   3. Build / typecheck (`QUALITY_COMMANDS.build` and/or `QUALITY_COMMANDS.typecheck`)
   4. Tests (`QUALITY_COMMANDS.test`) — at minimum the tests related to the current change
4. **Checkpoint commit**: commit the working state with a clear message describing what was implemented. Reference spec requirement identifiers if applicable.
5. **Mark completed** (TaskUpdate)

Do NOT wait until the end to run quality gates. Run them after each task to catch issues early.

### 3.4 Error recovery

When a quality gate fails:
1. **Read the error output** carefully
2. **Identify root cause** — is it a code bug, a missing import, a type error, a test assumption?
3. **Fix the issue**
4. **Re-run the full gate sequence** (format → lint → build → test) — don't just re-run the failing gate, as a fix might introduce new issues
5. **3-attempt limit**: if the same gate fails 3 times after different fix attempts, **STOP and escalate to the user** with:
   - The error output
   - What you've tried
   - Your analysis of the root cause
   - Suggested next steps

### 3.5 When to ask the user

**Be autonomous for tactical decisions:**
- Naming conventions (follow existing patterns)
- Import organization
- Code formatting (use the formatter)
- Implementation details within the scope of the spec
- Minor refactoring needed to integrate the new code

**Ask the user for:**
- Architectural decisions not covered by the spec
- Ambiguous spec requirements that could be interpreted multiple ways
- Dependency choices (e.g., "which HTTP client library?")
- Public API changes that go beyond spec scope
- Decisions that would be hard to reverse

---

## PHASE 4: VERIFICATION & TESTING

### 4.1 Implement new tests

Write tests for all new functionality:
- **Unit tests**: for individual functions and methods
- **Integration tests**: for component interactions
- **E2E tests**: for full user flows as specified in the spec (using whatever test identifiers the spec defines)
- Follow existing test patterns and conventions in `tests/`
- Cover: happy paths, failure paths, edge cases

### 4.2 Non-regression

- Run the **full existing test suite** to ensure nothing is broken
- If existing tests fail due to intentional changes (e.g., feature removal), update them accordingly
- If existing tests fail unexpectedly, fix the implementation — do NOT just fix the tests

### 4.3 Acceptance test loop (mandatory, 100% pass required)

A user story is NOT considered implemented until every acceptance test passes. Execute this loop:

1. Run the complete test suite using `QUALITY_COMMANDS.test` (or `QUALITY_COMMANDS.check` if it runs everything). **Always use the project's CI/CD chain (Makefile)**. Never run test files directly or use ad hoc commands.
2. Parse the output: count passed, failed, and errored tests.
3. **If any test fails or errors:**
   a. Analyze the failure(s)
   b. Fix the code (not the test expectations, unless the test itself is wrong)
   c. Run quality gates (format → lint → build)
   d. Go back to step 1
4. **Repeat until 0 failures and 0 errors.** Do NOT proceed to Phase 5, do NOT declare the story done, and do NOT ask the user to accept partial results while any acceptance test is failing.
5. Apply the 3 attempt limit from section 3.4 **per individual test**, not globally. If the same single test fails 3 times after different fix attempts, escalate that test to the user while continuing to fix others.

Only after 100% of tests pass, commit the final state and proceed.

---

## PHASE 5: DEPLOYMENT VERIFICATION

> Skip this phase if: there is no deploy target, the user explicitly opted out, or the project has no deployment infrastructure.

### 5.1 Deploy

If `QUALITY_COMMANDS.deploy` is available:
1. Run the deploy command
2. **If deployment fails**, enter a fix loop:
   - Analyze the error
   - If the fix is something you can do (code change, config fix, build issue) → fix it, re-run quality gates, commit, redeploy
   - If the fix requires **human action** (secret provisioning, manual config, IAM permissions, external service setup) → **STOP** and provide clear instructions:
     - What the user needs to do
     - Step-by-step commands if applicable
     - What to tell you when it's done so you can continue
   - Loop until deployment succeeds

### 5.2 Functional verification of new features

After deployment, verify all new functionality works end-to-end:
1. Use browser automation (via chrome-devtools MCP) or make API calls to test real behavior
2. Verify each spec requirement is actually working — not just passing unit tests
3. If human verification is needed (visual checks, OAuth flows, payment integrations) → tell the user what to check and what the expected behavior should be

### 5.3 Non-regression verification

Verify pre-existing functionality still works:
1. Run the full test suite again post-deployment
2. Spot-check critical existing features via browser/API
3. If anything broke → fix, re-run quality gates, commit, redeploy (loop back to 5.1)

### 5.4 Human-blocked items

Maintain a running list of items that require human action throughout this phase. At the end:
- Present them clearly with step-by-step instructions (e.g., "Deploy secret X to environment Y using command Z")
- Distinguish between blocking items (must be done before the feature works) and non-blocking items (can be done later)
- Continue with what can be automated, then summarize remaining manual steps

---

## PHASE 6: DOCUMENTATION & FINALIZATION

### 6.1 Update documentation

Update all project documentation to reflect the changes:

1. **README.md** — update usage, installation, configuration sections if affected
2. **CLAUDE.md** — update project overview, key commands, conventions if affected
3. **.agent_docs/*.md** — update or create topic files for new architecture, patterns, APIs
4. **docs/** — update user-facing documentation if it exists

Keep documentation updates proportional to the changes — a small feature addition doesn't need a full docs rewrite.

### 6.2 Sync with base branch

1. Fetch latest: `git fetch origin`
2. Rebase onto base branch: `git rebase origin/main` (or the appropriate base branch)
3. If conflicts arise, resolve them
4. If rebased (commits were replayed), re-run `QUALITY_COMMANDS.test` to ensure nothing broke from the rebase

### 6.3 Archive the spec (if implementing from a spec file)

- Create `specs/archived/` if it doesn't exist: `mkdir -p specs/archived/`
- Move the spec file: `git mv specs/<file> specs/archived/<file>`

### 6.4 Final commit

Commit documentation and archive changes. The commit message must:
- Summarize what was updated (docs, spec archive)
- Reference the spec file if applicable

### 6.5 Summary

Provide a clear summary:
- What was implemented (features, modifications, removals)
- Tests added/modified/removed and their pass status
- Documentation updated
- Spec archived (if applicable)
- Deployment status (if deployment was performed)
- **Manual steps still pending** (from Phase 5 human-blocked items, if any)
- Any items left as TBD or requiring follow-up

Tell the user they can use `/project:push` when ready to push and create a pull request.

---

## ITERATION MODE

After the initial implementation, the user may request tweaks, fixes, or small changes based on their own testing. This is normal and expected.

### Scope
Iteration mode is for:
- Bug fixes
- UI tweaks
- Config changes
- Small behavioral changes
- Copy/text updates

**Out of scope** for iteration mode (suggest a new `/project:implement` run instead):
- New features that weren't in the original spec
- Multi-module refactors
- Architectural changes

### For each iteration:
1. Make the requested change
2. Run quality gates: format → lint → build → test
3. Commit the change with a clear message (reference the spec if relevant)
4. If deployed: redeploy and re-verify after the change (follow Phase 5 flow)

Do NOT batch iteration changes — commit each one individually so the user has clean history.
error: cannot format -: Cannot parse for target version Python 3.14: 1:10: Implement changes in the current project, either from a specification file, a backlog entry, or a direct description.
