Conduct a thorough change discovery interview and generate a complete change specification for the current project.

**Input:** $ARGUMENTS

---

## GIT PRE-FLIGHT (MANDATORY)

Before starting any work:

1. Run `git status` — if the working tree is dirty (uncommitted changes), **STOP immediately** and tell the user to commit or stash before proceeding.
2. If git is not initialized, **STOP immediately** and tell the user to first initialize git. Don't initialize yourself.
3. Run `git branch --show-current` to identify the current branch.

### Context validation & branch setup
- If on a `spec/` branch → continue (iterating on specification). Verify remote sync:
  - `git fetch origin && git rev-list --count HEAD..origin/<current-branch>` (ignore errors if remote doesn't exist)
  - If count > 0 → **STOP**: "Branch is behind remote. Resolve manually."
- If on a `feat/` or `clean/` branch → **STOP**: "You are in a `{branch}` branch which is meant for implementation/compliance work, not spec writing. Switch to the main branch first."
- If on a protected branch (`main`, `master`, `uat`, `prod`, `production`, `preprod`, `preproduction`, `staging`, `tests`, `integration`, `qa`, `develop`) or any other branch → a new `spec/` branch will be created later (in Phase 4).

---

## PHASE 0: READ EXISTING PROJECT (MANDATORY)

Before any interview, you MUST read and understand the existing project. This phase is NOT optional.

Read the following in order, skipping any that don't exist:

### 0.1 Project Documentation
1. `CLAUDE.md` — project overview, conventions, key commands
2. `.agent_docs/*.md` — detailed architecture and patterns documentation
3. `README.md` — project description and usage
4. `docs/*` — user-facing documentation, design records

### 0.2 Project Configuration
5. `Makefile` — build, test, run, deploy commands
6. Configuration files (e.g., `pyproject.toml`, `go.mod`, `package.json`, `config.yaml`)

### 0.3 Existing Specifications
7. `specs/*` — all existing specification documents

Read every existing spec carefully. You MUST check consistency between the requested changes and existing specs. You CANNOT modify existing specs — but you MUST flag inconsistencies during the interview and document them in the change specification.

### 0.4 Source Code
Read the project source code at a depth appropriate to the change request:
- **Always read**: directory structure, entry points, main modules, public APIs
- **Read deeper** into areas that are likely affected by the requested changes
- **Skim** areas that are clearly unrelated

Use your judgment — a UI change doesn't require deep reading of database code, but a data model change requires reading everything that touches that model.

### 0.5 Tests
8. `tests/` — existing end-to-end and unit tests

Understand the current test coverage and patterns.

### 0.6 Synthesis

After reading, produce a brief internal summary (do NOT output this to the user yet) covering:
- What the project does
- Key architectural patterns
- Existing specs and their scope
- Current test coverage
- Areas relevant to the upcoming changes

---

## PHASE 1: FOCUSED INTERVIEW

Conduct a **focused, adaptive interview** using the **AskUserQuestion** tool. This is NOT a full discovery — the project already exists. Focus only on understanding the changes.

### Interview Structure

The interview has **2 rounds**, adaptive based on complexity.

#### Round 1: Change Understanding
Ask about (one question at a time, adapt based on answers):
- What modifications do you want to make? Classify each as:
  - **Add**: a new feature or capability
  - **Remove**: an existing feature to be eliminated
  - **Change**: modifying how an existing feature works (new constraints, different behavior, new dependencies)
- For each modification: what exactly, and why?
- What is the expected outcome? What does "done" look like?
- Are there constraints or dependencies for these changes?
- What is the priority if there are multiple modifications?

#### Round 2: Details & Edge Cases
For each modification, ask targeted questions:

**For additions:**
- What are the inputs, outputs, and business rules?
- How does this integrate with existing features?
- What error handling is expected?
- What edge cases do you foresee?

**For removals:**
- What should happen to features that depend on this?
- Should any data be preserved, migrated, or deleted?
- Are there external consumers (APIs, integrations) that will be affected?

**For changes:**
- What is the current behavior that must change?
- What is the new expected behavior?
- What triggered this change (performance issue, new constraint, dependency change)?
- Are there backward compatibility concerns?

#### Consistency Check
If you identified inconsistencies between the requested changes and existing specs during Phase 0:
- **Flag each inconsistency explicitly** to the user during the interview
- Ask how the user wants to resolve it
- Document the user's decision in the change specification
- Remind the user that existing specs will NOT be modified — the change spec will document the deviation

### Interview Rules
- Ask **one focused question at a time** using the AskUserQuestion tool
- **Prefer multiple choice questions** — use AskUserQuestion options to propose 2-4 choices rather than open-ended questions. Open-ended is fine when choices can't be anticipated.
- Be concise — the user already knows their project, don't re-explain what you read
- Probe deeper only when answers are ambiguous or incomplete
- **Apply YAGNI ruthlessly** — challenge scope creep during changes. If the user wants to add something that isn't essential to the change, ask "Is this essential for this change, or can we defer it?" If deferred, collect for the backlog.
- Summarize all modifications at the end and ask for final confirmation
- If the user says "skip" or "not sure", note it as TBD but move on
- Aim for 8-15 questions total, adapting based on change complexity
- Be efficient — this is a change spec, not a full discovery

---

## PHASE 2: APPROACH EXPLORATION

After the interview, propose **2-3 implementation strategies** for the requested changes. Do NOT skip this — even seemingly straightforward changes benefit from considering alternatives.

### Process
1. Present 2-3 distinct strategies for implementing the changes (e.g., incremental migration vs big-bang, different integration patterns, different levels of refactoring)
2. For each strategy, describe:
   - **Summary**: What this strategy looks like in 1-2 sentences
   - **Pros**: Key advantages
   - **Cons**: Key disadvantages and risks
   - **Affected scope**: How much of the codebase this strategy touches
   - **Best suited when**: Under what circumstances this strategy shines
3. **Lead with your recommended strategy** and explain why you recommend it
4. Use **AskUserQuestion** with the strategies as multiple choice options to let the user pick

### Example
For adding OAuth to a project with email/password auth:
- **Strategy A (Recommended)**: Add OAuth alongside existing auth — minimal disruption, users can use either method, incremental rollout
- **Strategy B**: Replace email/password entirely with OAuth — cleaner result, but breaking change for existing users, needs migration
- **Strategy C**: OAuth as a separate auth service — decoupled, but adds infrastructure complexity

The user's choice informs the impact analysis, requirements, and test design.

---

## PHASE 3: IMPACT ANALYSIS

After the user approves a strategy, perform a systematic impact analysis. This is analytical work — no user interaction needed.

### 3.1 Affected Components
Identify every source file, module, and package that will need changes. Be specific — list file paths.

### 3.2 Affected Requirements
Cross-reference with existing specs in `specs/`:
- Which existing FRs are impacted?
- Which existing NFRs are impacted?
- Are any existing requirements invalidated?

### 3.3 Affected Tests
Review `tests/` to identify:
- Tests that will need modification
- Tests that will need removal
- Test gaps (existing features not covered that will be affected)

### 3.4 Affected Documentation
Identify what documentation needs updating:
- `CLAUDE.md` and `.agent_docs/*.md` sections
- `docs/*` files
- `README.md` sections
- Inline code documentation

### 3.5 Dependencies & Risks
- New external dependencies needed
- Dependencies that can be removed
- Breaking changes for consumers
- Migration or data transformation needs
- Rollback strategy considerations

---

## PHASE 4: CHANGE SPECIFICATION GENERATION

### Step 1: Suggest Spec Title

Propose a short, descriptive title for the change spec and ask the user to confirm using **AskUserQuestion**:
- Derive a slug from the change description (e.g., "add-oauth-support", "remove-email-notifications", "migrate-to-mongodb")
- The final filename will be: `specs/<YYYY-mm-dd_HH:MM:SS>-<title>.md`
- Use the Bash tool with `date +%Y-%m-%d_%H:%M:%S` to get the current timestamp

### Step 2: Create spec branch

If not already on a `spec/` branch, create one:
1. `git checkout -b spec/<slug>`
2. All subsequent file operations happen on this branch

### Step 3: Create the specs/ directory

Use the Bash tool: `mkdir -p specs/`

### Step 4: Generate the change specification

**Scale each section to its actual complexity.** A small feature addition doesn't need the same depth as a database migration. Write a few sentences for straightforward sections, and up to 200-300 words for nuanced ones. The template below is a maximum structure — adapt it to the change's real complexity.

Generate the specification file with this structure:

```markdown
# [Change Title] — Change Specification

> Generated on: [date]
> Project: [project name]
> Version: 1.0
> Status: Draft
> Type: Change Specification

## 1. Change Summary
[Brief description of what changes are being requested and why — 2-3 sentences max]

### Modifications Overview
| MOD ID | Type | Title | Priority |
|--------|------|-------|----------|
[Table summarizing all modifications]

## 2. Current State Analysis

### 2.1 Project Overview
[Summary of the current project based on codebase reading]

### 2.2 Existing Specifications
[Summary of existing specs in specs/ — list each with a one-line description]

### 2.3 Relevant Architecture
[Current architectural elements that will be affected by the changes]

## 3. Requested Modifications

### MOD-001: [Modification Title]
- **Type:** Add / Remove / Change
- **Description:** [What needs to happen]
- **Rationale:** [Why this change is needed]
- **Priority:** Must-have / Should-have / Nice-to-have
- **Details:**
  - [Inputs, outputs, business rules for additions]
  - [Current vs new behavior for changes]
  - [Cleanup requirements for removals]
[Repeat for each modification]

## 4. Impact Analysis

### 4.1 Affected Components
| File/Module | Impact Type | Description |
|-------------|-------------|-------------|
[Table of affected source files with what changes are needed]

### 4.2 Affected Requirements
| Spec File | Requirement ID | Impact | Description |
|-----------|----------------|--------|-------------|
[Table cross-referencing existing spec requirements that are impacted]

### 4.3 Affected Tests
| Test File | Test ID/Name | Action | Description |
|-----------|-------------|--------|-------------|
[Table of existing tests that need modification, removal, or highlight gaps]

### 4.4 Affected Documentation
| Document | Section | Action | Description |
|----------|---------|--------|-------------|
[Table of documentation that needs updating]

### 4.5 Dependencies & Risks
[New dependencies, removed dependencies, breaking changes, migration needs, rollback considerations]

## 5. New & Modified Requirements

### New Requirements
#### FR-NEW-001: [Requirement Title]
- **Description:** [What the system must do]
- **Inputs:** [What data comes in]
- **Outputs:** [What the system produces]
- **Business Rules:** [Constraints and logic]
- **Priority:** [Must-have / Should-have / Nice-to-have]
[Repeat for each new requirement]

### Modified Requirements
#### FR-MOD-001: [Requirement Title] (references [original spec] FR-XXX)
- **Original behavior:** [What it did before]
- **New behavior:** [What it must do now]
- **Reason for change:** [Why]
- **Business Rules:** [Updated constraints]
[Repeat for each modified requirement]

### Removed Requirements
#### FR-DEL-001: [Requirement Title] (references [original spec] FR-XXX)
- **Description:** [What is being removed]
- **Reason:** [Why]
- **Cleanup:** [What code, data, config must be cleaned up]
[Repeat for each removed requirement]

## 6. Non-Functional Requirements Changes
[Only NFRs that are new or changed — reference existing spec NFRs where applicable]

## 7. Documentation Updates

All documentation changes listed below MUST be implemented as part of this change.

### 7.1 CLAUDE.md & .agent_docs/
[Specific sections and content that must be added, modified, or removed]

### 7.2 docs/*
[Specific documentation files and sections that must be updated]

### 7.3 README.md
[Specific sections that must be updated — installation, usage, configuration, etc.]

## 8. End-to-End Test Updates

All test changes MUST be implemented in the `tests/` directory. Every modification MUST have tests covering happy paths, failure paths, edge cases, and error recovery.

### 8.1 Test Summary
| Test ID | Action | Category | Scenario | Priority |
|---------|--------|----------|----------|----------|
[Table: Action = New / Modified / Removed]

### 8.2 New Tests
#### E2E-NEW-001: [Test Name]
- **Category:** [Core Journey / Feature / Error / Security / Performance]
- **Modification:** [Which MOD-XXX this validates]
- **Preconditions:** [Setup required]
- **Steps:**
  - Given [initial state]
  - When [action]
  - Then [expected outcome]
- **Priority:** [Critical / High / Medium / Low]
[Repeat for each new test — MUST include happy path, failure, and edge case tests for every new feature]

### 8.3 Modified Tests
#### E2E-MOD-001: [Test Name] (was [original test reference])
- **Original test:** [What it validated before]
- **Modified to validate:** [What it validates now]
- **Steps:**
  - Given [initial state]
  - When [action]
  - Then [expected outcome]
[Repeat for each modified test]

### 8.4 Removed Tests
#### E2E-DEL-001: [Test Name] (was [original test reference])
- **Reason:** [Why this test is no longer needed]
[Repeat for each removed test]

## 9. Consistency Notes
[Any inconsistencies found with existing specs, how they were discussed with the user, and the resolution decisions made]

## 10. Migration & Implementation Notes
[Specific implementation guidance, suggested order of operations, feature flags, data migration steps, rollback strategy]

## 11. Open Questions & TBDs
[Items that need further clarification]
```

### Step 5: Commit and summary

1. Commit the spec file: `git add specs/ && git commit -m "Add change specification: <title>"`
2. If deferred ideas exist, also commit `specs/BACKLOG.md`
3. Provide a brief summary of what was generated (counts of modifications, new/modified/removed requirements, tests, affected components, and deferred ideas sent to backlog)
4. Tell the user they can use `/project:push` when ready to push and create a pull request.

---

## QUALITY ASSURANCE CHECKLIST

Before delivering the final change specification, verify:
- [ ] 2-3 implementation strategies were proposed and user chose one before spec generation
- [ ] YAGNI was applied — no unnecessary scope creep; deferred ideas written to `specs/BACKLOG.md`
- [ ] Every requested modification has a MOD-XXX entry with clear type (Add/Remove/Change)
- [ ] Impact analysis covers all affected components, requirements, tests, and documentation
- [ ] Every new feature has E2E tests for happy path, failure path, AND edge cases
- [ ] Every modified feature has updated E2E tests
- [ ] Every removed feature has its tests marked for removal
- [ ] No ambiguous language (avoid "should", "might", "could" — use "must", "will")
- [ ] All new/modified/removed requirements have unique IDs with clear cross-references
- [ ] All new/modified/removed tests have unique IDs
- [ ] Documentation updates (CLAUDE.md, .agent_docs/, docs/*, README.md) are explicitly specified
- [ ] Section depth is proportional to complexity — no padded sections
- [ ] Inconsistencies with existing specs are documented with resolution decisions
- [ ] Existing specs are NOT modified — only referenced
- [ ] Open questions are clearly flagged as TBDs
- [ ] The spec file is written to `specs/<timestamp>-<title>.md`

---

## BEHAVIORAL RULES

1. **Always read the existing project first.** Phase 0 is mandatory — never skip codebase analysis.
2. **Always use AskUserQuestion tool** for the interview — do not make assumptions.
3. **Prefer multiple choice questions** — use AskUserQuestion options whenever possible to make the interview faster and more focused.
4. **Never modify existing specs.** Only reference them. Flag inconsistencies and document resolutions.
5. **Be focused** — this is a change spec, not a full project discovery. Ask only what's needed.
6. **Be precise** — specifications must be unambiguous and testable.
7. **Be thorough on impact** — the value of a change spec is in the impact analysis. Miss nothing.
8. **Apply YAGNI ruthlessly** — challenge scope creep. If the user agrees to defer an idea, write it to `specs/BACKLOG.md` (create the file if it doesn't exist, append if it does). Each backlog entry must have: ID, idea title, brief description, rationale for deferral, and which spec suggested it.
9. **Always propose 2-3 implementation strategies** — never jump straight from interview to impact analysis. Present alternatives with trade-offs and your recommendation.
10. **Scale sections to complexity** — adapt the depth of each spec section to its actual complexity. Don't pad simple sections.
11. **Present a summary of modifications** after the interview and before generating the spec, asking for final confirmation.
12. **Number everything** — all modifications, requirements, and tests must have traceable IDs with clear cross-references to existing specs.
13. **Write the spec file** to `specs/<YYYY-mm-dd_HH:MM:SS>-<title>.md` — never to the project root, never overwrite existing specs.
14. **Suggest a spec title** and ask the user to confirm before writing.
15. **After writing the specification**, provide a brief summary of what was generated (counts of modifications, new/modified/removed requirements, tests, affected components, and deferred ideas sent to backlog).
16. **Flag all inconsistencies** with existing specs during the interview — do not silently ignore them.
