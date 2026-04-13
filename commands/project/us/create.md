# Create User Story

Create a single, LLM ready user story through a structured brainstorming session with the user. This command is used when you need to create one story: from a spec entry point (an FR, a scenario, or a feature area), from a backlog entry, or from a fresh idea that does not yet have a spec.

**Critical principle:** A user story is a **vertical slice**, not a single requirement. When the input is a spec reference (FR-XXX, SC-XXX, or a feature keyword), the command analyzes the surrounding context and proposes an intelligent grouping of related requirements into a cohesive story. A good story contains 2 to 4 tightly coupled FRs, not one FR in isolation.

**Input:** $ARGUMENTS (an FR-XXX or SC-XXX ID, a feature keyword like "authentication", a backlog reference, a text description, or nothing for interactive mode)

**Output:** A single markdown file containing the complete, self contained user story ready for a coding agent.

---

## PHASE 1: UNDERSTAND INPUT

### 1.1 Determine input type

- **FR-XXX, SC-XXX, or feature keyword:** Search the provided spec for matching IDs or keyword. The input is an **entry point**, not necessarily the full scope of the story. Set `MODE = from_spec`.
- **Backlog reference** ("from backlog", "BL-NNN"): Locate the backlog entry. Set `MODE = from_backlog`.
- **Text description:** The user describes a feature in plain language. Set `MODE = from_scratch`.
- **No argument / interactive:** Ask the user what they want to build. Set `MODE = from_scratch`.

### 1.2 Load available context

Gather whatever context is available. All items are optional; use what exists:

1. The parent specification document (if referenced or provided)
2. Project documentation (`CLAUDE.md`, `.agent_docs/*.md`, `README.md`)
3. Project configuration (`Makefile`, `pyproject.toml`, `go.mod`, `package.json`)
4. Existing user stories (to check for overlap and determine the next US-XXX ID)
5. Source code structure and test directory layout

---

## PHASE 2: SCOPE DEFINITION

### 2.1 When MODE = from_spec

The input (FR-XXX, SC-XXX, or keyword) is an **entry point**, not the story boundary. A single FR is almost never a good story on its own. The goal is to identify a cohesive vertical slice around that entry point.

#### Step 1: Find the entry point in the spec

- If FR-XXX → locate that requirement
- If SC-XXX → locate that scenario and all its associated FRs
- If keyword → search the spec for matching sections, present findings, ask the user to confirm the area of interest

#### Step 2: Analyze the neighborhood

Starting from the entry point, expand outward using these cohesion criteria:

1. **Same scenario:** pull in all FRs that belong to the same SC-XXX. A scenario is a natural story boundary because it represents a complete user journey.
2. **Same data entity:** if the entry FR operates on an entity (e.g., "Order"), pull in other FRs that create, read, update, or delete that same entity. CRUD on one entity is a natural vertical slice.
3. **Direct dependencies:** if the entry FR depends on another FR (e.g., FR-003 "place order" requires FR-001 "create product catalog entry"), pull in the dependency if it is not already covered by an existing story.
4. **Shared file impact:** using the spec's impact analysis or your own reading of the codebase, identify FRs that modify the same files or modules. Co-located changes reduce merge conflicts and contextual overhead.
5. **Shared E2E tests:** if two FRs are validated by the same E2E test, they belong together.

#### Step 3: Apply sizing guard rails

After expansion, check the candidate grouping:

- **2 to 4 FRs:** ideal size. Proceed.
- **4 to 5 FRs:** acceptable if they are tightly coupled (same entity CRUD, or single scenario). Ask the user: "This groups 4 to 5 requirements. Should I keep them together or split?"
- **6+ FRs:** too large. Split into sub slices using the same cohesion criteria, and suggest creating multiple stories. Ask the user which slice to build first.
- **Only 1 FR, and it is trivial** (e.g., a config change, a simple validation rule): consider whether it should be folded into an adjacent story rather than standing alone. Ask the user: "This is a small change. Should it be its own story, or should I include it in a related story?"

#### Step 4: Check for overlap with existing stories

If existing user stories are available:
- If any of the candidate FRs are already assigned to an existing story → exclude them and inform the user
- If the candidate grouping overlaps significantly with an existing story → suggest extending that story instead of creating a new one

#### Step 5: Present the proposed grouping

Show the user:

1. **Proposed scope:** the list of FRs with their titles, grouped by cohesion reason
2. **Why these belong together:** explain the cohesion (same scenario, same entity, dependency chain, etc.)
3. **Associated scenario(s):** the SC-XXX IDs covered
4. **Associated E2E tests:** count and list of test IDs
5. **What is NOT included:** nearby FRs that were considered but excluded, with reason

Ask: "Does this grouping make sense as a single story? Want to add or remove anything?"

Iterate until the user confirms the scope. Then proceed to Phase 2.5 (Generate Tests).

### 2.2 When MODE = from_backlog

The backlog entry is usually terse. Expand it through brainstorming:

1. Show the backlog entry to the user
2. Proceed to the brainstorming interview (2.4)

### 2.3 When MODE = from_scratch

No existing context. Full brainstorming needed. Proceed to 2.4.

### 2.4 Brainstorming Interview

Conduct a focused interview using **AskUserQuestion**. The goal is to extract enough information to write a complete user story. This is NOT a full spec interview (that is `spec.md`'s job). This is targeted at a single feature slice.

#### Round 1: Intent and Scope (3 to 5 questions)

- What does this feature do in one sentence?
- Who uses it? (Actor/persona)
- What triggers it? (User action, API call, scheduled job, event)
- What is the expected outcome? (What changes in the system?)
- What is explicitly NOT included? (Scope boundary)

After Round 1, present a one paragraph summary: "So the goal is to [summary]. Is this accurate?"

#### Round 2: Behavior and Rules (3 to 5 questions)

- Walk through the happy path step by step: "What happens first? Then what? Then what?"
- What are the inputs? Ask for concrete examples with real values.
- What are the outputs? Ask for exact response shapes, status codes, or UI states.
- What business rules apply? (validation, authorization, limits, conditions)
- What side effects happen? (database writes, events published, notifications sent, caches invalidated)

After Round 2, present the behavior as a structured scenario (Actor / Preconditions / Flow / Postconditions) and confirm.

#### Round 3: Failure Modes (2 to 4 questions)

- What are the 3 most likely things to go wrong?
- What happens on invalid input? (ask for specific invalid cases)
- What happens when a dependency is unavailable?
- Are there authorization or permission failures to handle?

After Round 3, present the failure scenarios and confirm.

#### Round 4: Technical Fit (2 to 3 questions)

- Which existing files/modules will this touch?
- Are there similar features already implemented that this should follow as a pattern?
- Any new dependencies needed?
- Any files or areas that must NOT be modified?

### 2.5 Generate Tests

Based on the brainstorming or spec analysis, generate E2E tests BEFORE writing the story. This forces precision.

#### Step 1: Build the test plan

Present the test plan to the user:

1. List each test with: ID, category (happy/failure/edge/side-effect), one line description
2. Show counts: "N happy path, N failure, N edge case, N side effect"
3. Verify the failure to happy ratio is > 1:1
4. **Edge cases and error tests are mandatory.** For every happy path, identify at least:
   - 1 invalid input test (wrong type, missing field, out of range)
   - 1 boundary condition test (empty list, zero value, max length)
   - 1 authorization/permission failure test (if applicable)
   - Each edge/error test must specify the **exact error response** expected (error code, message, HTTP status). A test that says "should fail" without specifying how is incomplete.
5. Ask: "Does this test plan cover everything? Any scenarios missing?"

Iterate until the user confirms.

#### Step 2: Identify test data requirements

For each test, determine what data is needed (fixtures, seed records, mock responses, configuration):

1. List all required test data with: description, format, source (generated/provided/fixture)
2. For each data item, determine:
   - **Can it be auto generated?** (e.g., random users, synthetic records, factory fixtures) → propose the generation strategy and ask: "I plan to generate this data automatically. Does this approach work?"
   - **Must the user provide it?** (e.g., real API keys, specific business data, production samples, credentials) → ask the user explicitly: "This test requires [data]. Can you provide it?"
3. Present a data summary table: test ID, data needed, source (auto/user), status (ready/pending)
4. Do NOT proceed to story assembly until all test data sources are confirmed. Missing data = incomplete story.

---

## PHASE 3: STORY ASSEMBLY

### 3.1 Determine story ID

- If existing stories are available, find the highest US-XXX ID and assign the next sequential one
- If no existing stories, start with US-001

### 3.2 Derive the slug

From the story title, create a kebab case slug: "OAuth Client Credentials" → `oauth-client-credentials`

### 3.3 Assemble technical context

From available project context:

1. **Stack:** extract from project documentation and configuration files
2. **File structure:** the directories and files this story will affect
3. **Existing patterns:** if similar functionality exists, extract the pattern (function signatures, error handling style, naming conventions)
4. **Data model:** extract relevant entities from existing code or spec

### 3.4 Write the story file

Generate `US-XXX_<slug>.md` using this template:

```markdown
# US-XXX: [Title]

> Parent Spec: [spec reference, or "standalone" if from scratch]
> Status: draft
> Priority: [TBD — user sets this]
> Depends On: [US-YYY or "none"]
> Complexity: [S / M / L — estimate based on FR count and test count]

## Objective

[2 to 3 sentences: what user problem this solves, what capability the system gains]

## Technical Context

### Stack
[language, framework, key libraries, naming conventions]

### Relevant File Structure
```
[partial tree showing ONLY files relevant to this story]
```

### Existing Patterns
[how similar features are implemented, with file references]

### Data Model (excerpt)
[only entities this story creates or modifies, with fields, types, constraints]

## Functional Requirements

### FR-XXX: [Title]
[If from_spec: copy from spec with original ID]
[If from_scratch: assign new IDs starting from FR-US-XXX-001]
- **Description:** [what the system must do]
- **Inputs:** [with concrete examples]
- **Outputs:** [with concrete examples]
- **Business Rules:** [constraints and logic]

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
[If from_spec: use original IDs]
[If from_scratch: assign new IDs starting from E2E-US-XXX-001]
- **Category:** happy
- **Scenario:** [SC-XXX if from spec, or inline scenario description]
- **Requirements:** [FR refs]
- **Preconditions:**
  - [concrete setup data]
- **Steps:**
  - Given [concrete state]
  - When [exact action with exact inputs]
  - Then [exact outcome with exact values]
  - And [side effect verification]
- **Cleanup:** [teardown]
- **Priority:** [Critical / High / Medium / Low]

### Edge Case and Error Tests

> **Edge case and error tests are equally mandatory.** These tests verify that the system correctly rejects invalid inputs, enforces boundaries, and returns proper error responses. Each test MUST specify the **exact expected error** (error code, HTTP status, error message). A test that says "should fail" without specifying the exact error is incomplete and will be rejected.

### E2E-XXX: [Test Name]
- **Category:** [failure / edge]
- **Scenario:** [what goes wrong]
- **Requirements:** [FR refs]
- **Preconditions:**
  - [concrete setup data]
- **Steps:**
  - Given [concrete state]
  - When [invalid action with exact bad inputs]
  - Then [exact error: HTTP status, error code, error message]
  - And [system state unchanged / no side effects]
- **Cleanup:** [teardown]
- **Priority:** [Critical / High / Medium / Low]

[Repeat for each E2E test]

## Constraints

### Files Not to Touch
[paths that are out of scope]

### Dependencies Not to Add
[only these packages are allowed, or "no new dependencies"]

### Patterns to Avoid
[project specific anti patterns]

### Scope Boundary
[what is explicitly NOT part of this story even if related]

## Non Regression

### Existing Tests That Must Pass
[list test files or test names]

### Behaviors That Must Not Change
[specific behaviors to preserve]
```

---

## PHASE 4: REVIEW AND CONFIRM

Present the generated story to the user:

1. Show the full story content
2. Ask: "Does this story look complete and accurate? Anything to add, remove, or change?"
3. If changes are requested → apply them to the output
4. Ask: "Should I set the status to 'ready' for implementation?"

---

## BEHAVIORAL RULES

1. **Brainstorming is mandatory for from_scratch and from_backlog modes.** Do not skip the interview. The story quality depends on the questions asked.
2. **For from_spec mode, the spec provides the content but NOT the story boundary.** The input FR or SC is an entry point. Always analyze the neighborhood and propose a cohesive grouping of 2 to 4 FRs. Never create a story with a single trivial FR when related FRs form a natural vertical slice together.
3. **Tests before story.** Always generate and confirm the test plan before writing the story file. Test driven thinking produces better stories.
4. **Concrete examples everywhere.** Every input, output, and test assertion must use real values. No placeholders.
5. **One story per run.** This command creates one story at a time. For bulk splitting, use `spec2us`.
6. **Self contained output.** The generated story must be usable as a standalone prompt for the implementation command without needing any other document.
7. **Preserve spec IDs.** When creating from a spec, use the original FR-XXX and E2E-XXX IDs. When creating from scratch, use the US-scoped ID scheme (FR-US-XXX-NNN, E2E-US-XXX-NNN) to avoid collisions with future specs.
8. **Ask, do not assume.** When in doubt about scope, behavior, or constraints, ask the user. A question takes 10 seconds; a wrong assumption wastes an entire implementation cycle.
9. **Technical context is non negotiable.** Even for from_scratch stories, gather and embed the context. The agent that implements this story needs it.
10. **Vertical slices, not horizontal layers.** A story that creates a DB schema without the API that uses it is useless. A story that implements an API endpoint without the validation rules is incomplete. Always cut vertically through the stack.
11. **Prefer cohesion over minimalism.** When in doubt between a slightly larger story that is self contained and a smaller story that depends on unfinished work, choose the larger one. The LLM performs better with a complete picture.
12. **A story MUST leave the system in a working state.** For a new product: the implemented features work end to end, even if the product is incomplete. For an existing product: all pre-existing functionality continues to work without regression. A story that breaks existing behavior or produces a non-functional system is not done.
