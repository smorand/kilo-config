You will check all the compliance aspects of this project.

## Git Pre-flight (MANDATORY)

### Protected Branches
These branches must NEVER be committed to directly: `main`, `master`, `uat`, `prod`, `production`, `preprod`, `preproduction`, `staging`, `tests`, `integration`, `qa`, `develop`.

### Pre-flight & Branch Setup
1. Run `git status` — if the working tree is dirty, **STOP immediately** and ask the user to commit or stash before running compliance.
2. Run `git branch --show-current` to identify the current branch.
3. Branch management:
   - If on a protected branch → create: `git checkout -b clean/compliance`
   - If already on `clean/compliance` → verify remote sync:
     - `git fetch origin && git rev-list --count HEAD..origin/clean/compliance` (ignore errors if remote branch doesn't exist yet)
     - If count > 0 → **STOP**: "Branch `clean/compliance` is behind remote. Another developer may have pushed changes. Resolve manually."
     - If up to date → continue
   - If on a different non-protected branch → continue on it

## Discovery

1. Figure out if Terraform is used in this project (look for `.tf` files in an `iac/` folder)
2. Figure out what programming language is used: either Python or Go. If neither is detected, skip the language compliance check and inform the user.

## Compliance checks

Launch the following subagents in parallel using the Task tool (one subagent per check):

- **Programming language compliance:** Use `golang-compliance-checker` subagent if Go, or `python-skill-compliance` subagent if Python
- **Terraform compliance** (only if Terraform is used): Use `terraform-compliance-checker` subagent
- **End-to-end tests:** Use `e2e-test-writer` subagent
- **Project documentation:** Use `project-docs-writer` subagent

## Reporting

Once all subagents have completed, produce a **full detailed report** of every transformation each agent performed. This must include:

- **Per agent:** List every file created, modified, or deleted, with a description of what was changed and why
- **Status per check:** pass / fail / skipped
- **Key issues found**, grouped by category
- **Recommended next steps** to resolve any remaining failures

The report must be exhaustive — do not summarize or omit agent actions. The user expects to see the complete picture of all modifications made across the entire project.

## After Work

Commit all changes. Tell the user they can use `/project:push` when ready to push and create a pull request.
