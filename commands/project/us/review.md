# Review User Story

Review an existing user story for completeness, accuracy, and implementability. Fix issues, fill gaps, and iterate until the story is ready for a coding agent.

**Input:** $ARGUMENTS (a US-XXX ID, a path to a story file, a story's inline content, or "all" to review every available story)

**Output:** The corrected user story markdown file(s), with all issues resolved or flagged.

---

## PHASE 1: LOAD STORY

### 1.1 Locate the story

- **US-XXX ID:** find the story file by ID from available stories.
- **File path:** read the file.
- **Inline content:** use the provided text directly.
- **"all":** collect all available stories. Process each one sequentially.
- **No argument:** list all available stories with their status. Ask the user which one to review using **AskUserQuestion**.

### 1.2 Load context

Gather whatever context is available. All items are optional; use what exists:

1. Project documentation (`CLAUDE.md`, `.agent_docs/*.md`, `README.md`)
2. Project configuration (`Makefile`, `pyproject.toml`, `go.mod`, `package.json`)
3. The parent spec (if referenced in the story)
4. Other user stories (to check for dependency correctness and overlap)
5. Source code in areas the story touches
6. Test directory

---

## PHASE 2: AUTOMATED REVIEW

Run the following checks automatically. Collect all findings before presenting them.

### 2.1 Structural Completeness

Verify every required section exists and is non empty:

| Section | Check |
|---------|-------|
| Story Identity | ID, title, parent spec, status, priority, depends on, complexity |
| Objective | 2 to 3 sentences present, explains "why" |
| Technical Context: Stack | Language, framework, libraries listed |
| Technical Context: File Structure | Tree or file list present |
| Technical Context: Existing Patterns | At least one reference file cited |
| Technical Context: Data Model | Entities listed with fields and types |
| Functional Requirements | At least 1 FR with description, inputs, outputs, business rules |
| Acceptance Tests | At least 5 E2E tests |
| Constraints | At least one of: files not to touch, dependencies, patterns, scope |
| Non Regression | Existing tests or behaviors listed |

**Finding levels:**
- **MISSING:** section absent or empty → must fix
- **WEAK:** section present but lacks specificity → should improve
- **OK:** section meets requirements

### 2.2 Requirement Quality

For each functional requirement:

- [ ] Has concrete input examples with real values (not "valid input")
- [ ] Has concrete output examples with real values (not "returns success")
- [ ] Business rules are unambiguous (uses "must" not "should" or "might")
- [ ] No circular or self referencing rules
- [ ] Traceable to parent spec (if from_spec mode): FR-XXX ID matches the spec

### 2.3 Test Quality

- [ ] **Coverage:** every FR has at least 1 happy path test, 1 failure test, 1 edge case test
- [ ] **Ratio:** failure tests outnumber happy path tests (count and report the ratio)
- [ ] **Concreteness:** every Given/When/Then uses exact values, never placeholders
- [ ] **Side effects:** operations with side effects have explicit verification steps
- [ ] **Preconditions:** every test specifies concrete setup data
- [ ] **Cleanup:** tests that create data specify teardown
- [ ] **IDs:** every test has a unique ID, no duplicates across stories

### 2.4 Technical Context Accuracy

Compare the story's technical context against actual project state (if project context is available):

- [ ] **Stack matches reality:** the language/framework in the story matches project configuration
- [ ] **File structure is current:** the files listed actually exist (or are clearly marked as "to be created")
- [ ] **Patterns are real:** the reference files cited in "Existing Patterns" exist and demonstrate the claimed pattern
- [ ] **Data model is consistent:** entities in the story match existing data models (if any)

### 2.5 Dependency and Overlap Check

If other stories are available:

- [ ] **Dependencies are valid:** every US-YYY in "Depends On" exists
- [ ] **No circular dependencies:** follow the dependency chain and verify no cycles
- [ ] **No FR overlap:** no FR-XXX appears in more than one story
- [ ] **No test overlap:** no E2E-XXX appears in more than one story
- [ ] **Scope boundaries are respected:** the story's constraints do not conflict with another story's scope

### 2.6 Implementability Check

- [ ] **Self contained:** could an agent implement this story using ONLY this file plus the project's documentation? If the story references external information without including it, flag it.
- [ ] **Scope is achievable:** the story does not try to do too much (more than 4 FRs or 15 tests suggests splitting)
- [ ] **Scope is sufficient:** the story is not too thin. A single trivial FR (one validation rule, one config change) should usually be folded into a related story. Flag stories with fewer than 2 FRs or fewer than 5 tests.
- [ ] **Vertical slice, not horizontal layer:** the story covers data model through API/UI for its feature, not just one layer. A story that only creates models or only adds endpoints without the other layers is incomplete.
- [ ] **No ambiguity traps:** no requirements that could be interpreted multiple ways without additional context
- [ ] **Working state guarantee:** after implementation, the system must be functional. For a new product: implemented features work end to end, even if the product is incomplete. For an existing product: no regression on pre-existing functionality. If the story lacks a Non Regression section (existing product) or does not produce a runnable feature (new product), flag it.

---

## PHASE 3: PRESENT FINDINGS

### 3.1 Summary dashboard

Present a review summary:

```
Story: US-XXX — [Title]
Overall: [READY / NEEDS WORK / MAJOR ISSUES]

Structural Completeness:  [X/10 sections OK]
Requirement Quality:      [X/N requirements pass all checks]
Test Quality:             [X/N tests pass all checks, ratio: 1:Y]
Technical Accuracy:       [X/N checks pass]
Dependencies:             [OK / issues found]
Implementability:         [OK / concerns]

Issues Found: [N total — M must-fix, K should-improve]
```

### 3.2 Detailed findings

For each issue, present:
- **Section:** where the issue is
- **Level:** MUST FIX / SHOULD IMPROVE / SUGGESTION
- **Finding:** what is wrong
- **Recommendation:** how to fix it

Group by level (MUST FIX first, then SHOULD IMPROVE, then SUGGESTION).

### 3.3 User decision

Ask the user using **AskUserQuestion**:
- "Fix all issues automatically" → proceed to Phase 4
- "Fix must-fix issues only" → proceed to Phase 4 with reduced scope
- "Let me review the findings first" → wait for user input on each issue
- "Story is fine as is, mark as ready" → skip to Phase 5

---

## PHASE 4: FIX AND ITERATE

### 4.1 Apply fixes

For each issue to fix:

1. If the fix requires user input (ambiguous requirement, missing business rule), ask using **AskUserQuestion**
2. If the fix can be derived from the project context (outdated file structure, missing stack info), apply automatically
3. If the fix requires reading additional source code, read it first

### 4.2 Regenerate tests if needed

If requirement changes affected the test plan:
1. Regenerate or update the affected tests
2. Re verify the failure to happy ratio
3. Present the updated test plan for confirmation

### 4.3 Update the story output

Write the corrected story, overwriting the previous version.

### 4.4 Re run automated review

After applying fixes, run Phase 2 again to verify all issues are resolved. Present any remaining issues.

---

## PHASE 5: FINALIZE

### 5.1 Update status

If the story passes all checks, set status to `ready` in the output file.

### 5.2 Summary

Present:
- Issues found and fixed
- Remaining issues (if user chose partial fix)
- Current story status
- Suggest next step: implementation

---

## BATCH MODE (when input = "all")

When reviewing all stories:

1. Run Phase 2 on every story, collect all findings
2. Present a consolidated dashboard:

```
Story Review Summary
====================
| ID | Title | Status | Issues | Must Fix | Verdict |
|----|-------|--------|--------|----------|---------|
| US-001 | [title] | ready | 2 | 0 | READY |
| US-002 | [title] | draft | 5 | 3 | NEEDS WORK |
| US-003 | [title] | ready | 1 | 1 | NEEDS WORK |
```

3. Ask the user: "Which stories should I fix? All with issues, only must-fix, or let me pick?"
4. Process selected stories sequentially through Phase 3 to 5

---

## BEHAVIORAL RULES

1. **Automated checks first, always.** Run Phase 2 before asking the user anything. Findings drive the conversation.
2. **Be specific in findings.** "Test E2E-003 uses placeholder 'valid input' instead of concrete data" is actionable. "Tests could be improved" is not.
3. **Fix what you can, ask what you cannot.** If the project context provides the answer, fix it silently. If it requires a judgment call, ask.
4. **Do not weaken the story.** Fixes must improve specificity, never reduce it. Do not simplify tests or remove assertions to "fix" coverage issues.
5. **Maintain traceability.** Fixes must not break the ID scheme. If adding tests, continue the existing numbering.
6. **The failure to happy ratio is sacred.** If the ratio drops below 1:1 after changes, add more failure tests.
7. **Technical context must match reality.** If the codebase has changed since the story was written, update the context to reflect the current state.
8. **Cross story consistency matters.** In batch mode, verify that stories do not overlap, contradict each other, or have broken dependency chains.
9. **Do not over review.** A story that passes all automated checks and has user confirmation is done. Perfectionism is the enemy of shipping.
