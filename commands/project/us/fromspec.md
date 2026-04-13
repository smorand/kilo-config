# Split Specification into User Stories

Split a specification document into well scoped, LLM ready user stories for iterative implementation by a coding agent.

**Input:** $ARGUMENTS (a specification document: file path, inline content, or "latest" to use the most recent spec in the project)

**Output:** A set of user story markdown files, one per story, plus an index file summarizing the full backlog.

---

## PHASE 1: LOAD CONTEXT

### 1.1 Load the specification

- If a path is given → read the file.
- If "latest" or no argument → find the most recent specification document available, read it.
- If no spec exists → **STOP**: "No specification found. Create a specification first."

### 1.2 Load project context

Gather whatever context is available. All items are optional; use what exists:

1. Project documentation (`CLAUDE.md`, `.agent_docs/*.md`, `README.md`)
2. Project configuration (`Makefile`, `pyproject.toml`, `go.mod`, `package.json`)
3. Existing user stories (to avoid duplicating already-covered FRs)
4. Source code structure (language-appropriate file listing)
5. Test directory structure

---

## PHASE 2: SPEC ANALYSIS

### 2.1 Extract the spec skeleton

From the specification, extract and organize:

1. **All Scenarios** (SC-XXX) with their flows and exceptions
2. **All Functional Requirements** (FR-XXX / FR-NEW-XXX / FR-MOD-XXX) with their dependencies
3. **All E2E Tests** (E2E-XXX) mapped to their scenarios and requirements
4. **The Traceability Matrix** (if present in the spec)
5. **Impact Analysis** (if existing project) — which files/modules are affected

### 2.2 Build the dependency graph

Map relationships between functional requirements:

- Which FRs depend on other FRs? (e.g., FR-003 "create order" depends on FR-001 "create product")
- Which FRs share data entities?
- Which FRs share API surface area?
- Which FRs modify the same files?

This graph determines the story boundaries and implementation order.

### 2.3 Identify natural vertical slices

Group FRs into candidate stories using these heuristics (in priority order):

1. **Scenario cohesion:** FRs that belong to the same scenario (SC-XXX) tend to form a natural story. A complete scenario is often the ideal story boundary.
2. **Data entity cohesion:** FRs that operate on the same entity (CRUD operations) group together. "Create product", "validate product fields", and "store product image" are one story, not three.
3. **Dependency chains:** FRs with linear dependencies form a story in dependency order. If FR-003 cannot exist without FR-001, they belong together unless the combined size is too large.
4. **File impact overlap:** FRs that modify the same set of files belong together. Co-located changes reduce merge risk and give the LLM a coherent picture.
5. **Shared test coverage:** FRs that are validated by the same E2E tests should be in the same story. Splitting them forces the test to span stories, which breaks the "self contained" rule.

**Anti-pattern: 1 FR = 1 story.** Never create a story with a single FR unless that FR is genuinely large and self contained (a complex algorithm, a full integration with an external system). Most individual FRs are too thin: they produce fragments that do not form a working feature. The LLM performs best when it sees the complete vertical slice.

**Anti-pattern: horizontal layers as stories.** Never create stories like "US-001: all database models", "US-002: all API endpoints", "US-003: add all validation". These horizontal slices prevent the LLM from building a coherent feature. Always slice vertically: one story handles models + endpoint + validation + tests for one feature area.

**Sizing guard rails:**
- A story SHOULD contain 2 to 4 FRs. A single FR story is a red flag; 5+ FRs suggest splitting.
- A story SHOULD map to 1 to 2 scenarios. If it spans 3+, it is too broad.
- A story SHOULD have 5 to 15 E2E tests. Fewer means it is too thin. More means it is too thick.
- A story MUST be implementable in a single implementation run without exceeding context limits.
- A story MUST leave the system in a working state: after implementation, the feature it covers works end to end and all tests pass.

### 2.4 Determine implementation order

Order the stories by:

1. **Dependencies first:** if US-002 depends on US-001, US-001 comes first
2. **Foundation before features:** data models and core APIs before complex business logic
3. **Happy paths before edge cases:** get the basic flow working, then harden it
4. **High priority FRs first:** respect the priority from the spec (Must-have before Should-have)

---

## PHASE 3: USER REVIEW

### 3.1 Present the slicing plan

Present to the user using **AskUserQuestion** where appropriate:

1. **Story map overview:** a table showing each proposed story with:
   - Proposed ID (US-001, US-002, ...)
   - Proposed title
   - FRs included (by ID)
   - Scenarios covered (by ID)
   - E2E test count
   - Dependencies on other stories
   - Estimated complexity (S / M / L)

2. **Implementation order:** the sequence with rationale

3. **Dependency diagram** (text based):
   ```
   US-001 (Auth setup)
     └──▶ US-002 (User management)
     └──▶ US-003 (API endpoints)
            └──▶ US-004 (Business logic)
                   └──▶ US-005 (Error handling & hardening)
   ```

### 3.2 Collect feedback

Ask the user:
- "Does this slicing make sense? Would you merge or split any stories?"
- "Is the implementation order correct? Any story you want to prioritize differently?"
- "Are there any stories you want to skip for now and defer to the backlog?"

Iterate on the plan until the user confirms.

---

## PHASE 4: GENERATE USER STORIES

### 4.1 Generate each user story file

For each confirmed story, generate a markdown file `US-XXX_<slug>.md` using this template:

```markdown
# US-XXX: [Title]

> Parent Spec: [spec reference]
> Status: ready
> Priority: [1-N, implementation order]
> Depends On: [US-YYY, US-ZZZ or "none"]
> Complexity: [S / M / L]

## Objective

[2 to 3 sentences: what user problem this solves, what capability the system gains]

## Technical Context

### Stack
[Language, framework, key libraries — from project context]

### Relevant File Structure
```
[Partial tree output showing ONLY the files relevant to this story]
```

### Existing Patterns
[How similar functionality is implemented in the codebase today. Reference specific files.
If greenfield: reference the architectural decisions from the spec.]

### Data Model (excerpt)
[Only the entities this story creates or modifies. Include field names, types, constraints.]

## Functional Requirements

### FR-XXX: [Title]
- **Description:** [from spec]
- **Inputs:** [with concrete examples]
- **Outputs:** [with concrete examples]
- **Business Rules:** [from spec]

[Repeat for each FR in this story]

## Acceptance Tests

> **Acceptance tests are mandatory: 100% must pass.**
> A user story is NOT considered implemented until **every single acceptance test below passes**.
> The implementing agent MUST loop (fix code → run tests → check results → repeat) until all acceptance tests pass with zero failures. Do not stop or declare the story "done" while any test is failing.
> Tests MUST be validated through the project's CI/CD chain (generally `make test` or equivalent Makefile target). No other method of running or validating tests is acceptable: do not run test files directly, do not use ad hoc commands. Use the Makefile.

### Test Data

> If tests require specific data (fixtures, seed records, configuration, mock responses), it MUST be documented here. Auto generated data must be described with its generation strategy. User provided data must be flagged and collected before implementation begins.

| Data | Description | Source | Status |
|------|-------------|--------|--------|
| [data name] | [what it is and which tests use it] | [auto-generated / user-provided / fixture] | [ready / pending] |

[Repeat for each data item]

### Happy Path Tests

### E2E-XXX: [Test Name]
- **Category:** happy
- **Scenario:** SC-XXX
- **Requirements:** FR-XXX
- **Preconditions:**
  - [concrete data setup]
- **Steps:**
  - Given [state with concrete data]
  - When [action with exact inputs]
  - Then [outcome with exact values]
  - And [side effect verification]
- **Cleanup:** [teardown]
- **Priority:** [Critical / High / Medium / Low]

### Edge Case and Error Tests

> **Edge case and error tests are equally mandatory.** These tests verify that the system correctly rejects invalid inputs, enforces boundaries, and returns proper error responses. Each test MUST specify the **exact expected error** (error code, HTTP status, error message). A test that says "should fail" without specifying the exact error is incomplete and will be rejected.

### E2E-XXX: [Test Name]
- **Category:** [failure / edge]
- **Scenario:** [what goes wrong]
- **Requirements:** FR-XXX
- **Preconditions:**
  - [concrete data setup]
- **Steps:**
  - Given [state with concrete data]
  - When [invalid action with exact bad inputs]
  - Then [exact error: HTTP status, error code, error message]
  - And [system state unchanged / no side effects]
- **Cleanup:** [teardown]
- **Priority:** [Critical / High / Medium / Low]

[Repeat for each E2E test assigned to this story]

## Constraints

### Files Not to Touch
- [list of paths that are out of scope]

### Dependencies Not to Add
- [only these packages are allowed: list, or "no new dependencies"]

### Patterns to Avoid
- [anti patterns for this codebase]

### Scope Boundary
- [what is explicitly NOT part of this story even if related]

## Non Regression

### Existing Tests That Must Pass
- [list test files or test names]

### Behaviors That Must Not Change
- [list specific behaviors to preserve]

### API Contracts to Preserve
- [list endpoints and their current signatures, if applicable]
```

### 4.2 Generate the index file

Create an index file `_index.md`:

```markdown
# User Stories Index

> Source Specification: [spec reference]
> Generated on: [date]
> Total Stories: [N]

## Implementation Order

| Order | ID | Title | FRs | Scenarios | Tests | Depends On | Complexity | Status |
|-------|-----|-------|-----|-----------|-------|------------|------------|--------|
| 1 | US-001 | [title] | FR-001, FR-002 | SC-001 | 8 | — | M | ready |
| 2 | US-002 | [title] | FR-003 | SC-002 | 6 | US-001 | S | ready |
[...]

## Dependency Graph

```
[text based dependency diagram]
```

## Traceability

Every FR from the spec MUST appear in exactly one user story.
Every E2E test from the spec MUST appear in exactly one user story.
Every scenario from the spec MUST be covered by at least one user story.

### Coverage Verification
- FRs in spec: [N] | FRs assigned to stories: [N] | Unassigned: [list or "none"]
- E2E tests in spec: [N] | Tests assigned to stories: [N] | Unassigned: [list or "none"]
- Scenarios in spec: [N] | Scenarios covered: [N] | Uncovered: [list or "none"]
```

---

## PHASE 5: SUMMARY

Present:
1. Number of stories generated
2. Implementation order with estimated complexity
3. Total E2E test coverage
4. Any FRs or tests that were hard to assign (and how they were resolved)
5. Suggest starting implementation with the first story in order

---

## BEHAVIORAL RULES

1. **Every FR from the spec must appear in exactly one user story.** No orphans, no duplicates.
2. **Every E2E test from the spec must appear in exactly one user story.** The test belongs to the story that implements the behavior it validates.
3. **Stories must be self contained.** An agent reading a single story file must have enough context to implement it without reading the full spec.
4. **Prefer fewer, thicker stories over many thin ones.** LLMs work better with complete vertical slices than micro tasks.
5. **Technical context is mandatory, never optional.** The agent cannot infer conventions.
6. **Concrete examples in every requirement and test.** No placeholders, no vague assertions.
7. **The first story must be implementable independently.** It has no dependencies on other stories.
8. **Each story must leave the system in a working state.** After implementing any story, the full test suite must pass.
9. **Ask the user before generating.** The slicing plan must be confirmed. Do not generate stories without user approval of the plan.
10. **Use the spec's ID scheme.** FR-XXX, SC-XXX, E2E-XXX IDs from the spec must be preserved in the stories for traceability.
