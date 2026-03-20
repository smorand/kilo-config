Conduct a thorough requirements discovery interview and generate a complete specification for a NEW project.

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

## PHASE 0: PROJECT CONTEXT (OPTIONAL)

Before starting the interview, check if the project directory already contains code or documentation. If it does, ask the user whether they want you to consider existing project files to inform the specification.

Use the **AskUserQuestion** tool to ask:
- "This project already has existing files. Should I read and consider them (CLAUDE.md, docs/, source code, Makefile, tests/) to inform the specification, or should I treat this as a fresh start?"

If the user says **yes**, read the following (if they exist):
1. `CLAUDE.md` and `.agent_docs/*.md` — project conventions and architecture
2. `docs/*` — existing documentation
3. `README.md` — project description
4. `Makefile` — build/run commands
5. Source code structure (directory listing, entry points, key modules)
6. `tests/` — existing test patterns
7. `specs/*` — existing specifications

Use this context to pre-fill known information and avoid redundant questions during the interview. Acknowledge what you found and focus the interview on gaps.

If the project directory is empty or the user says **no**, skip this phase and proceed directly to the interview.

---

## PHASE 1: DISCOVERY INTERVIEW

You MUST conduct a thorough, structured interview with the user using the **AskUserQuestion** tool. Do NOT skip this phase. Do NOT assume requirements. Do NOT generate specifications without completing the interview.

### Interview Structure

Conduct the interview in **4 distinct rounds**. After each round, synthesize what you've learned and confirm understanding before proceeding.

#### Round 1: Scope & Vision
Ask about (one question at a time, adapt based on answers):
- What is the application/project name?
- What problem does it solve? Who is it for?
- **What type of project is this?** Use AskUserQuestion with options: API, MCP Server, WebApp, Mobile App, CLI
- What is the high-level scope? What's included?
- What are explicit **non-goals** (things deliberately excluded)?
- What is the target timeline or MVP definition?
- Are there existing systems this replaces or integrates with?
- What is the programming language preference? (see defaults below)

##### Project Type Defaults

Apply these defaults automatically. Confirm with the user but do not re-ask if the default is obvious:

| Type | Default Language | Default Deployment | Default Frontend | Notes |
|------|-----|------|------|-------|
| API | Go | Cloud Run (GCP) | N/A | |
| MCP Server | Go | Cloud Run (GCP) | N/A | Ask: transport (stdio, SSE, streamable HTTP) |
| WebApp | Go | Cloud Run (GCP) | HTMX | Server-side rendering |
| Mobile App | Flutter | App Stores | N/A | Usually needs a backend API |
| CLI | Go | Binary releases | N/A | Local execution |

- **Language override**: If the user says Python or Rust, use that instead. If not specified, default to Go (except Mobile → Flutter).
- **Deployment override**: GCP is default for API/MCP/WebApp. Always confirm: "Deploying on GCP as usual, or somewhere else?"

#### Round 2: Usage Scenarios

This round has 3 sub-rounds. Be thorough — usage scenarios are the foundation for both requirements and tests.

##### 2a: Scenario Enumeration
First, collect the full list of scenarios before going deep:
- Who are the primary user personas/actors?
- What are the **main usage scenarios** (user journeys)? List them all.
- Are there admin/operator scenarios distinct from end-user scenarios?
- Are there background/system scenarios (scheduled jobs, triggers, data pipelines)?

Present the full scenario list back to the user and ask: "Are these all the scenarios, or did I miss any?"

##### 2b: Scenario Deep-Dive
For **each** scenario, walk through it step by step using AskUserQuestion:
- What does the user do at each step? What does the system respond?
- What are the preconditions (what must be true before this scenario starts)?
- What are the postconditions (what is true after it completes)?
- What can go wrong at each step? (probe explicitly — don't accept "nothing" without pushing back)
- **Cross-scenario interactions**: "If a user is mid-way through [Scenario A] and triggers [Scenario B], what happens?" Ask this for any scenarios that could realistically overlap.

After each scenario deep-dive, present it back in structured format (Actor / Preconditions / Flow / Postconditions / Exceptions) and ask the user to confirm or correct it before moving to the next.

##### 2c: Failure & Error Scenarios (dedicated sub-round)
After all happy-path scenarios are confirmed, systematically probe for failures:
- For each scenario: "What are the **3 most likely things to go wrong** in this flow?"
- What happens on invalid/malformed input at each step?
- What happens when external dependencies are unavailable (API down, DB unreachable, third-party timeout)?
- What happens on permission/authorization failures?
- What happens on concurrent/conflicting operations (two users doing the same thing)?
- Are there partial failure states (operation half-completed)? How should the system recover?
- Ask the user: "Is there anything else that could go wrong that we haven't covered?"

#### Round 3: Functional Requirements (FR)
Ask about:
- What are the core **features/behaviors** the system must have?
- For each feature: what are the inputs, outputs, and business rules?
- What data entities are involved? What are their relationships?
- Are there workflows or state machines (e.g., order states, approval flows)?
- What integrations are needed (APIs, databases, third-party services)?
- What are the authorization/permission rules?
- Are there any specific algorithms or business logic to implement?

#### Round 4: Non-Functional Requirements (NFR)
Ask about each category:
- **Performance**: Expected load, response times, throughput, data volumes?
- **Security & Authentication** (adapt questions based on project type from Round 1):
  - Data sensitivity, compliance requirements (GDPR, SOC2, etc.)?
  - **Authentication mechanism** (use AskUserQuestion with type-appropriate options):
    - **API**: API key, OAuth2/OIDC, service account, or behind gateway?
    - **MCP Server**: API key, OAuth2/OIDC, or unauthenticated (behind gateway)?
    - **WebApp**: IAP on Google Cloud (simplest for internal/corporate), OIDC (Google, Auth0, Firebase Auth, Okta), or custom login?
    - **Mobile App**: OAuth2 + PKCE (standard for mobile), Firebase Auth, or custom?
    - **CLI**: API key, OAuth2 device flow, or service account?
  - **Identity provider**: "Which identity provider?" (Google, Auth0, Firebase Auth, Okta, custom)
  - **OIDC configuration** (if applicable): "Who provides the client ID/secret? Where will they be configured?"
  - **Session management** (WebApp): cookie-based sessions, JWT, or other?
  - **Service-to-service auth**: "Does this service call other services? If so, how? (GCP service accounts, shared API keys, mTLS)"
  - **Credential & secret storage**:
    - For deployed services: "GCP Secret Manager as usual, or something else?" (Secret Manager is the default)
    - How injected at runtime: env vars at deploy time, SDK calls to Secret Manager, mounted volumes?
    - For Mobile: Flutter secure_storage (backed by Keychain on iOS, Keystore on Android)
    - For CLI: local config file (~/.config/appname/) — what secrets need local storage?
  - **Token/key lifecycle**: rotation policy, revocation mechanism?
- **Usability**: Accessibility standards, supported devices/browsers, i18n/l10n?
- **Reliability**: Uptime requirements, disaster recovery, data backup?
- **Observability**: Logging, monitoring, alerting, tracing requirements?
  - OpenTelemetry is **mandatory** — ask: "What collector should we use? Default is JSONL file, but OTLP (Jaeger, Grafana Tempo, etc.) is also supported."
  - Ask: "Are there LLM calls in this project? If so, we'll trace model, tokens, cost, and duration (never prompts/responses)."
  - Ask: "Are there specific sub-metrics or custom spans you'd like beyond the defaults (API calls, DB queries, errors, auth ops)?"
- **Deployment & Infrastructure**: Cloud provider, CI/CD expectations, environment strategy (dev/staging/prod)?
  - **Hosting**: "Where will this be deployed?" (Cloud Run, GKE, VM, serverless, etc.)
  - **Domain & Routing**: "Will this need a custom domain? Should we use a simple domain mapping or a full HTTPS Load Balancer (with CDN, WAF, custom headers)?" — For most APIs/MCPs, domain mapping is simpler and free; LB adds ~$18/month but enables advanced features.
  - **API Gateway**: "Will there be an API gateway in front of this service?" (e.g., Apigee, Cloud Endpoints, Kong, or direct access)
  - Ask: "Is there a pattern from your other deployed services we should follow for consistency?"
  - **Type-specific infrastructure questions**:
    - **API/MCP/WebApp** (deployed services):
      - **Scale-to-zero rule**: All infrastructure MUST support scale-to-zero. Never propose Cloud SQL or always-on instances. The only exception is Redis (Memorystore) for shared cache, but use sparingly and confirm need.
      - Database: "Do you need a database?"
        - For credentials/tokens/sessions/documents → **Firestore** (default, scale-to-zero)
        - For relational/SQL needs → **AlloyDB** (scale-to-zero capable)
        - Never propose Cloud SQL (does not scale to zero)
      - Cache: "Do you need shared caching?" → Memorystore (Redis) only if justified (exception to scale-to-zero rule, confirm the need). Otherwise prefer in-memory cache.
      - Object storage: "Do you need file/blob storage?" → GCS buckets?
    - **WebApp** (HTMX-specific):
      - Template engine: html/template (stdlib) or templ?
      - CSS framework: Tailwind, Pico, DaisyUI, or custom?
      - Static assets: served from Cloud Run directly, or CDN (GCS + Cloud CDN)?
    - **MCP Server**:
      - Transport: stdio (local dev), SSE, or streamable HTTP?
      - "Will this MCP server be used by Claude Desktop, Claude Code, or a custom client?"
    - **Mobile App**:
      - "Does this need a backend API?" → If yes, spec both together or separately?
      - Distribution: App Store, Google Play, both? Internal (Enterprise) distribution?
      - Push notifications needed?
      - Offline support requirements?
    - **CLI**:
      - Distribution: homebrew tap, binary releases (GitHub Releases), or both?
      - Configuration: ~/.config/appname/ (XDG-compliant)?
      - Shell completions needed? (bash, zsh, fish)
- **Scalability**: Growth expectations, horizontal/vertical scaling needs?
- **Maintainability**: Code quality standards, documentation expectations?

### Interview Rules
- Ask **one focused question at a time** using the AskUserQuestion tool
- **Prefer multiple choice questions** — use AskUserQuestion options whenever possible to make the interview faster and more focused.
- After receiving an answer, acknowledge it, probe deeper if needed, then move to next question
- If the user's answer is vague, ask clarifying follow-ups before moving on
- **Apply YAGNI ruthlessly** — challenge features that seem unnecessary for MVP. Ask "Is this essential for the first version?" If the user agrees to defer, collect the idea for the backlog.
- Summarize findings at the end of each round and ask for confirmation
- Keep track of all answers systematically
- If the user says "skip" or "not sure" for a topic, note it as TBD but move on
- Aim for 15-25 questions total across all rounds, adapting based on project complexity
- Be conversational but efficient — respect the user's time

---

## PHASE 2: APPROACH EXPLORATION

After completing the interview, propose **2-3 alternative approaches** before committing to a specification. Do NOT skip this phase — even "obvious" projects benefit from explicitly considering alternatives.

### Process
1. Present 2-3 distinct approaches (e.g., different architectures, tech stacks, data models, scope strategies)
2. For each approach, describe:
   - **Summary**: What this approach looks like in 1-2 sentences
   - **Pros**: Key advantages
   - **Cons**: Key disadvantages and risks
   - **Best suited when**: Under what circumstances this approach shines
3. **Lead with your recommended option** and explain why you recommend it
4. Use **AskUserQuestion** with the approaches as multiple choice options to let the user pick

### Example
For a new API project, you might propose:
- **Approach A (Recommended)**: Monolith with modular structure — simpler to start, easier to deploy, refactor later if needed
- **Approach B**: Microservices from the start — better separation, but more infra complexity for MVP
- **Approach C**: Serverless functions — lowest ops overhead, but vendor lock-in and cold start concerns

The user's choice informs the architecture, data model, and NFRs in the specification.

---

## PHASE 3: END-TO-END TEST GENERATION

After the user approves an approach, generate comprehensive end-to-end (E2E) tests BEFORE writing the specification. This ensures test-driven thinking.

### 3.1 E2E Test Design Principles
- Each test MUST trace back to a **specific usage scenario** from Round 2 (by scenario ID)
- Each test MUST trace forward to the **functional requirements** it validates (by FR ID)
- Tests must be written in **Gherkin-style** (Given/When/Then) for clarity
- Include setup/teardown descriptions
- Group tests by feature area
- Each test must have a unique identifier (e.g., E2E-001)
- Include both functional and non-functional validation tests

### 3.2 Mandatory Test Coverage

Every feature MUST have tests covering ALL of the following:
1. **Happy path**: The normal, expected flow works correctly
2. **Failure paths**: What happens when operations fail (invalid input, service unavailable, unauthorized access, timeouts, data corruption)
3. **Edge cases**: Boundary conditions, empty states, maximum values, concurrent access, special characters, Unicode, very long inputs
4. **Error recovery**: System behavior after failures (retry, rollback, graceful degradation)

### 3.3 Edge Case Taxonomy

For each feature, systematically consider these edge case categories (skip categories that don't apply):

| Category | Examples |
|----------|----------|
| **Empty states** | No data, first-time user, cleared/reset data, empty lists |
| **Boundary values** | Min/max inputs, 0, -1, character limits, pagination boundaries, off-by-one |
| **Concurrent actions** | Multiple users on same resource, rapid repeated clicks, race conditions |
| **Network conditions** | Slow responses, timeouts, connection drops, retries (if applicable) |
| **Auth states** | Logged in, logged out, expired session, different roles, token refresh |
| **Data variations** | Special characters, Unicode, very long strings, HTML/script injection, null/empty fields |
| **Navigation patterns** | Back button, direct URL access, deep linking, refresh mid-flow, browser history |
| **Partial operations** | Operation interrupted mid-way, partial data saved, rollback scenarios |
| **External dependency failures** | Third-party API down, rate limited, returning unexpected data |

### 3.4 Test Categories to Cover
1. **Core User Journeys**: Complete end-to-end flows for each primary scenario from Round 2
2. **Feature-Specific Tests**: Detailed tests for each functional requirement — happy path, failure, and edge cases
3. **Error Handling Tests**: Invalid inputs, service failures, network errors, race conditions, data validation failures
4. **Security Tests**: Authentication failures, authorization violations, injection attempts, data protection validation
5. **Performance Baseline Tests**: Response time expectations, load handling, resource limits
6. **Integration Tests**: Third-party service interactions, API contract validation, failure modes of external dependencies
7. **Cross-Scenario Tests**: Interactions between scenarios identified in Round 2b

### 3.5 User Review of Test Plan

Before proceeding to spec generation, present the complete test list to the user:
1. Show a summary table: Test ID, Category, Scenario it validates, Priority
2. Show the count by category (e.g., "12 happy path, 8 failure, 15 edge case, 4 security, 3 performance")
3. Ask: "Does this test plan cover everything you care about? Any scenarios or edge cases missing?"
4. If the user identifies gaps, add tests and re-present
5. Only proceed to Phase 4 after the user confirms the test plan

---

## PHASE 4: SPECIFICATION GENERATION

### Step 1: Suggest Spec Title

Before writing the specification, propose a short, descriptive title for the spec file and ask the user to confirm or change it using **AskUserQuestion**:
- Derive a slug from the project name (e.g., "task-management-app", "auth-microservice")
- The final filename will be: `specs/<YYYY-mm-dd_HH:MM:SS>-<title>.md`
- Use the Bash tool with `date +%Y-%m-%d_%H:%M:%S` to get the current timestamp

### Step 2: Create spec branch

If not already on a `spec/` branch, create one:
1. `git checkout -b spec/<slug>`
2. All subsequent file operations happen on this branch

### Step 3: Create the specs/ directory

Use the Bash tool to create the `specs/` directory if it doesn't exist: `mkdir -p specs/`

### Step 4: Generate the specification

**Scale each section to its actual complexity.** A simple project doesn't need a 10-row NFR table or a detailed data model diagram. Write a few sentences for straightforward sections, and up to 200-300 words for nuanced ones. The template below is a maximum structure — adapt it to the project's real complexity.

Generate the specification file with this structure:

```markdown
# [Project Name] — Specification Document

> Generated on: [date]
> Version: 1.0
> Status: Draft

## 1. Executive Summary
[Brief description of the project, its purpose, and target audience]

## 2. Scope
### 2.1 In Scope
[Bulleted list of what's included]
### 2.2 Out of Scope (Non-Goals)
[Bulleted list of what's explicitly excluded]

## 3. User Personas & Actors
[Description of each user type/role]

## 4. Usage Scenarios
### SC-001: [Scenario Name]
**Actor:** [Who]
**Preconditions:** [What must be true before]
**Flow:**
1. [Step-by-step user journey — what the user does AND what the system responds at each step]
**Postconditions:** [What is true after successful completion]
**Exceptions:**
- [EXC-001a]: [What can go wrong at step N] → [How the system handles it]
- [EXC-001b]: [Another failure mode] → [System response]
**Cross-scenario notes:** [Interactions with other scenarios, if any]
[Repeat for each scenario]

## 5. Functional Requirements
### FR-001: [Requirement Title]
- **Description:** [What the system must do]
- **Inputs:** [What data comes in]
- **Outputs:** [What the system produces]
- **Business Rules:** [Constraints and logic]
- **Priority:** [Must-have / Should-have / Nice-to-have]
[Repeat for each requirement]

## 6. Non-Functional Requirements
### 6.1 Performance
[Specific, measurable requirements]
### 6.2 Security
[Authentication, authorization, data protection]
### 6.3 Usability
[UX standards, accessibility]
### 6.4 Reliability
[Uptime, recovery, backup]
### 6.5 Observability
[OpenTelemetry is mandatory. Specify:]
- **Collector**: [JSONL file (default) | OTLP to Jaeger/Tempo/etc.]
- **Trace file path**: [e.g., `traces/app.jsonl`]
- **What to trace**: API calls (INFO), DB queries (DEBUG), external tool calls (INFO), file mutations (DEBUG), auth ops (INFO), errors (ERROR), warnings (WARNING)
- **LLM tracing** (if applicable): model, token counts, duration, cost — NEVER prompts/responses
- **Custom sub-metrics**: [project-specific spans beyond defaults]
- **Sensitive data exclusion**: LLM prompts/responses, credentials, tokens, PII — NEVER in traces
[Additional logging, monitoring, alerting requirements]
### 6.6 Deployment
[Infrastructure, CI/CD, environments]
### 6.7 Scalability
[Growth and scaling strategy]

## 7. Data Model
[Key entities and their relationships — can be a table or diagram description]

## 8. Documentation Requirements

All documentation listed below MUST be created and maintained as part of this project.

### 8.1 README.md
- Project description, purpose, and audience
- Prerequisites and installation instructions
- How to run, build, and test the project
- Configuration and environment variables
- Usage examples

### 8.2 CLAUDE.md & .agent_docs/
- `CLAUDE.md`: Compact index with project overview, key commands, essential conventions, and documentation index referencing `.agent_docs/` files
- `.agent_docs/*.md`: Detailed documentation organized by topic (architecture, API, data model, deployment, etc.)
- Must be kept in sync with code changes

### 8.3 docs/*
- User-facing documentation (guides, tutorials, API reference)
- Architecture and design decision records
- Operational runbooks (if applicable)

## 9. Traceability Matrix

| Scenario | Functional Req | E2E Tests (Happy) | E2E Tests (Failure) | E2E Tests (Edge) |
|----------|---------------|-------------------|---------------------|-------------------|
| SC-001 | FR-001, FR-002 | E2E-001 | E2E-005, E2E-006 | E2E-010 |
[Every scenario MUST have at least one test in each column. Every FR MUST appear at least once.]

## 10. End-to-End Test Suite

All tests MUST be implemented in the `tests/` directory. Each feature MUST have tests covering happy paths, failure paths, edge cases, and error recovery.

### 10.1 Test Summary
| Test ID | Category | Scenario | FR refs | Priority |
|---------|----------|----------|---------|----------|
[Table of all tests — each test must link to both a scenario (SC-XXX) and requirement(s) (FR-XXX)]

### 10.2 Test Specifications
#### E2E-001: [Test Name]
- **Category:** [Core Journey / Feature / Error / Security / Performance / Cross-Scenario]
- **Scenario:** SC-XXX — [Which usage scenario this validates]
- **Requirements:** FR-XXX, FR-YYY — [Which functional requirements this covers]
- **Preconditions:** [Setup required]
- **Steps:**
  - Given [initial state]
  - When [action]
  - Then [expected outcome]
- **Priority:** [Critical / High / Medium / Low]
[Repeat for each test]

## 11. Open Questions & TBDs
[Items that need further clarification]

## 12. Glossary
[Domain-specific terms and definitions]
```

### Step 5: Commit and summary

1. Commit the spec file: `git add specs/ && git commit -m "Add specification: <title>"`
2. If deferred ideas exist, also commit `specs/BACKLOG.md`
3. Provide a brief summary of what was generated (counts of requirements, tests, scenarios, documentation requirements, and deferred ideas sent to backlog)
4. Tell the user they can use `/project:push` when ready to push and create a pull request.

---

## QUALITY ASSURANCE CHECKLIST

Before delivering the final specification, verify:

### Interview & Process
- [ ] 2-3 approaches were proposed and user chose one before spec generation
- [ ] YAGNI was applied — no unnecessary features included; deferred ideas written to `specs/BACKLOG.md`
- [ ] Every usage scenario was confirmed with the user in structured format (Actor/Preconditions/Flow/Postconditions/Exceptions)
- [ ] Failure/error scenarios were explicitly probed for each scenario (Round 2c)
- [ ] Test plan was reviewed and approved by the user before spec generation (Phase 3.5)

### Traceability (Section 9 — the matrix must be complete)
- [ ] Every scenario (SC-XXX) has at least one E2E test for happy path, failure path, AND edge cases
- [ ] Every functional requirement (FR-XXX) is covered by at least one E2E test
- [ ] Every E2E test references both its scenario (SC-XXX) and its requirements (FR-XXX)
- [ ] No orphan tests (tests that don't link to a scenario or requirement)
- [ ] No orphan requirements (FRs with no test coverage)
- [ ] No orphan scenarios (scenarios with no E2E test)

### Spec Quality
- [ ] Every non-functional requirement has measurable acceptance criteria
- [ ] No ambiguous language (avoid "should", "might", "could" — use "must", "will")
- [ ] All requirements have unique IDs (FR-XXX)
- [ ] All scenarios have unique IDs (SC-XXX)
- [ ] All tests have unique IDs (E2E-XXX)
- [ ] Documentation requirements (README.md, CLAUDE.md, .agent_docs/, docs/*) are specified
- [ ] Section depth is proportional to complexity — no padded sections
- [ ] Open questions are clearly flagged as TBDs
- [ ] The document is self-contained and understandable without additional context
- [ ] The spec file is written to `specs/<timestamp>-<title>.md`

---

## BEHAVIORAL RULES

1. **Never skip the interview phase.** Even if the user provides upfront context, validate and probe deeper.
2. **Always use AskUserQuestion tool** for the interview — do not make assumptions.
3. **Prefer multiple choice questions** — use AskUserQuestion options whenever possible to make the interview faster and more focused.
4. **Be methodical** — follow the interview rounds, then propose approaches, then generate.
5. **Be precise** — specifications must be unambiguous and testable.
6. **Be complete** — cover happy paths, failure modes, AND edge cases.
7. **Apply YAGNI ruthlessly** — actively challenge features that aren't essential for MVP. If the user agrees to defer an idea, write it to `specs/BACKLOG.md` (create the file if it doesn't exist, append if it does). Each backlog entry must have: ID, idea title, brief description, rationale for deferral, and which spec suggested it.
8. **Always propose 2-3 approaches** — never jump straight from interview to specification. Present alternatives with trade-offs and your recommendation.
9. **Scale sections to complexity** — adapt the depth of each spec section to its actual complexity. Don't pad simple sections.
10. **Present a summary of findings** after the interview and before generating tests/specs, asking for final confirmation.
11. **Number everything** — all requirements (FR-XXX), scenarios (SC-XXX), and tests (E2E-XXX) must have traceable IDs.
12. **Write the spec file** to `specs/<YYYY-mm-dd_HH:MM:SS>-<title>.md` — never to the project root.
13. **Suggest a spec title** and ask the user to confirm before writing.
14. **After writing the specification**, provide a brief summary of what was generated (counts of requirements, tests, scenarios, documentation requirements, and deferred ideas sent to backlog).
15. **Ask about existing project context** if project files exist — never ignore them without asking.
16. **Confirm each scenario individually** — after deep-diving into a scenario, present it in structured format and get user confirmation before moving to the next.
17. **Probe failures explicitly** — never accept "nothing can go wrong" without pushing back. Dedicate Round 2c entirely to failure and error scenarios.
18. **Get user sign-off on the test plan** — present the full test list (Phase 3.5) and only proceed to spec generation after the user approves it.
19. **Build the traceability matrix** (Section 9) — every scenario must link to FRs and E2E tests. No orphans allowed.
