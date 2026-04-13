Conduct a thorough requirements discovery interview and generate a complete specification for the current project.

This command handles both **new projects** (greenfield) and **existing projects** (adding features, evolving the system). It automatically detects the project state and adapts its behavior accordingly.

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
- If on a protected branch (`main`, `master`, `uat`, `prod`, `production`, `preprod`, `preproduction`, `staging`, `tests`, `integration`, `qa`, `develop`) or any other branch → a new `spec/` branch will be created later (in Phase 5).

---

## PHASE 0: PROJECT CONTEXT (MANDATORY)

Before starting the interview, you MUST analyze the project directory to determine its state. This phase is NOT optional.

### 0.1 Detect Project State

Check for the presence of code, documentation, and specs. Classify the project as:

- **Greenfield**: The project directory is empty or contains only boilerplate (e.g., just a `.git/`, empty `README.md`, no source code). → Set `PROJECT_MODE = greenfield`
- **Existing**: The project has source code, configuration, specs, or meaningful documentation. → Set `PROJECT_MODE = existing`

### 0.2 Read Existing Project (when PROJECT_MODE = existing)

Read the following in order, skipping any that don't exist:

#### Project Documentation
1. `CLAUDE.md` and `.agent_docs/*.md` — project conventions and architecture
2. `docs/*` — existing documentation
3. `README.md` — project description

#### Project Configuration
4. `Makefile` — build, test, run, deploy commands
5. Configuration files (e.g., `pyproject.toml`, `go.mod`, `package.json`, `config.yaml`)

#### Existing Specifications
6. `specs/*` — all existing specification documents

Read every existing spec carefully. You MUST check consistency between the new specification and existing specs. You CANNOT modify existing specs — but you MUST flag inconsistencies during the interview and document them in the specification.

#### Source Code
7. Read the project source code at a depth appropriate to the specification request:
   - **Always read**: directory structure, entry points, main modules, public APIs
   - **Read deeper** into areas likely affected by the new specification
   - **Skim** areas that are clearly unrelated

#### Tests
8. `tests/` — existing end-to-end and unit tests. Understand current coverage and patterns.

#### Synthesis

After reading, produce a brief internal summary (do NOT output this to the user yet) covering:
- What the project does
- Key architectural patterns
- Existing specs and their scope
- Current test coverage
- Areas relevant to the upcoming specification

Use this context to pre-fill known information and avoid redundant questions during the interview. Acknowledge what you found and focus the interview on gaps.

### 0.3 Greenfield Projects (when PROJECT_MODE = greenfield)

No reading needed. Proceed directly to the interview.

---

## PHASE 1: DISCOVERY INTERVIEW

You MUST conduct a thorough, structured interview with the user using the **AskUserQuestion** tool. Do NOT skip this phase. Do NOT assume requirements. Do NOT generate specifications without completing the interview.

### Interview Structure

Conduct the interview in **4 distinct rounds**. After each round, synthesize what you've learned and confirm understanding before proceeding.

**Adaptive depth**: When `PROJECT_MODE = existing`, skip questions whose answers are already clear from Phase 0. Acknowledge what you already know ("From the codebase, I can see X — is that still accurate?") and focus on what's new or changing.

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

**When `PROJECT_MODE = existing`**: Many of these answers are already known. Confirm what you read and focus on what's new: "The project is currently a Go API deployed on Cloud Run. What new capabilities are you looking to add?"

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

**When `PROJECT_MODE = existing`**: List the scenarios you identified from the existing codebase/specs first. Then ask: "These are the existing scenarios I found. Which new scenarios are you adding? Are any existing scenarios changing?"

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

**When `PROJECT_MODE = existing`**: For existing scenarios that are not changing, do NOT deep-dive again. Only deep-dive new scenarios and scenarios that are being modified.

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

**When `PROJECT_MODE = existing`**: Focus on new and modified requirements. For existing FRs that are unchanged, reference them by ID from existing specs rather than re-specifying.

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

**When `PROJECT_MODE = existing`**: Skip NFR categories that are already established and not changing. Only ask about NFRs that are new or impacted by the new specification.

#### Consistency Check (when PROJECT_MODE = existing)

If you identified inconsistencies between the new specification and existing specs during Phase 0:
- **Flag each inconsistency explicitly** to the user during the interview
- Ask how the user wants to resolve it
- Document the user's decision in the specification
- Remind the user that existing specs will NOT be modified — the new spec will document the deviation

### Interview Rules
- Ask **one focused question at a time** using the AskUserQuestion tool
- **Prefer multiple choice questions** — use AskUserQuestion options whenever possible to make the interview faster and more focused.
- After receiving an answer, acknowledge it, probe deeper if needed, then move to next question
- If the user's answer is vague, ask clarifying follow-ups before moving on
- **Apply YAGNI ruthlessly** — challenge features that seem unnecessary for MVP. Ask "Is this essential for the first version?" If the user agrees to defer, collect the idea for the backlog.
- Summarize findings at the end of each round and ask for confirmation
- Keep track of all answers systematically
- If the user says "skip" or "not sure" for a topic, note it as TBD but move on
- **Greenfield**: Aim for 15-25 questions total across all rounds, adapting based on project complexity
- **Existing**: Aim for 8-15 questions total, focusing on what's new or changing
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
   - **Affected scope**: How much of the codebase this approach touches (for existing projects: list key files/modules impacted)
   - **Best suited when**: Under what circumstances this approach shines
3. **Lead with your recommended option** and explain why you recommend it
4. Use **AskUserQuestion** with the approaches as multiple choice options to let the user pick

### Example (greenfield)
For a new API project, you might propose:
- **Approach A (Recommended)**: Monolith with modular structure — simpler to start, easier to deploy, refactor later if needed
- **Approach B**: Microservices from the start — better separation, but more infra complexity for MVP
- **Approach C**: Serverless functions — lowest ops overhead, but vendor lock-in and cold start concerns

### Example (existing project)
For adding OAuth to a project with email/password auth:
- **Approach A (Recommended)**: Add OAuth alongside existing auth — minimal disruption, users can use either method, incremental rollout
- **Approach B**: Replace email/password entirely with OAuth — cleaner result, but breaking change for existing users, needs migration
- **Approach C**: OAuth as a separate auth service — decoupled, but adds infrastructure complexity

The user's choice informs the architecture, data model, and NFRs in the specification.

---

## PHASE 3: IMPACT ANALYSIS (when PROJECT_MODE = existing)

**Skip this phase entirely for greenfield projects.**

After the user approves an approach, perform a systematic impact analysis. This is analytical work — no user interaction needed.

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

## PHASE 4: END-TO-END TEST GENERATION

> **E2E tests are the MOST IMPORTANT deliverable of this specification.** They are the primary contract that guides agent-based implementation. An agent implementing code will use these tests as its success criteria: code is done when all E2E tests pass. Unit tests are secondary and can be derived during implementation. E2E tests define *what the system must do* from the outside, covering working scenarios, non-working scenarios, side effects, state transitions, edge cases, and integration boundaries. Invest maximum effort here.

After the user approves an approach (and after impact analysis for existing projects), generate comprehensive end-to-end (E2E) tests BEFORE writing the specification. This ensures test-driven thinking.

### 4.1 E2E Test Design Principles

**Tests must be implementable by an agent without ambiguity.** Each test is a contract: precise enough that an agent reading it can write the test code AND the production code to make it pass.

- Each test MUST trace back to a **specific usage scenario** from Round 2 (by scenario ID)
- Each test MUST trace forward to the **functional requirements** it validates (by FR ID)
- Tests must be written in **Gherkin-style** (Given/When/Then) for clarity
- **Every "Then" step must assert something concrete and observable** — no vague outcomes like "the system handles it correctly". Specify exact status codes, response bodies, error messages, state changes, or side effects.
- **Specify exact test data** — don't say "valid input"; say `{"name": "Alice", "email": "alice@example.com"}`. Don't say "invalid input"; say `{"name": "", "email": "not-an-email"}`.
- **Specify the verification method** — how does the test confirm the outcome? API response check, database query, file existence, log entry, event emission, or external service call?
- Include setup/teardown descriptions with specific data and state
- Group tests by feature area
- Each test must have a unique identifier (e.g., E2E-001)
- Include both functional and non-functional validation tests
- **When `PROJECT_MODE = existing`**: Also identify tests that must be **modified** or **removed** due to the changes

### 4.2 Mandatory Test Coverage

Every feature MUST have tests covering ALL of the following categories. Do NOT settle for just happy path tests.

1. **Happy path (working scenarios)**: The normal, expected flow works correctly with valid inputs and produces the expected outputs AND side effects
2. **Failure paths (non-working scenarios)**: What happens when operations fail:
   - Invalid/malformed input (wrong types, missing required fields, out-of-range values)
   - Unauthorized access (missing auth, wrong role, expired token)
   - Service unavailable (dependency down, timeout, connection refused)
   - Business rule violations (duplicate entries, insufficient balance, quota exceeded)
   - Data corruption or inconsistency
3. **Side effects verification**: After an operation completes, verify ALL observable consequences:
   - Data created, modified, or deleted in the database/store
   - Events/messages published to queues or topics
   - Notifications sent (email, push, webhook)
   - Files created, modified, or deleted
   - Cache entries created, invalidated, or updated
   - Audit log entries written
   - Counters/metrics incremented
   - Related entities updated (cascading changes)
4. **State transitions**: For any entity with a lifecycle (statuses, workflows), test:
   - Every valid state transition (e.g., draft → published, pending → approved)
   - Every invalid state transition (e.g., published → draft must fail, or completed → pending must fail)
   - The system's state after each transition (what changed, what didn't)
5. **Edge cases**: Boundary conditions, empty states, maximum values, concurrent access, special characters, Unicode, very long inputs
6. **Error recovery**: System behavior after failures (retry, rollback, graceful degradation, idempotency)
7. **Idempotency**: For operations that should be idempotent, verify that repeating the same operation produces the same result without duplication or corruption

### 4.3 Edge Case Taxonomy

For each feature, systematically walk through these categories and generate tests for every applicable case. **Do NOT skip categories without explicitly confirming they don't apply.**

| Category | Examples | What to assert |
|----------|----------|----------------|
| **Empty states** | No data, first-time user, cleared/reset data, empty lists, empty strings, null values | Graceful handling, appropriate defaults, correct empty-state response |
| **Boundary values** | Min/max inputs, 0, -1, character limits, pagination boundaries, off-by-one, max int, empty collection vs single item vs many | Exact behavior at each boundary, correct error for out-of-bounds |
| **Concurrent actions** | Multiple users on same resource, rapid repeated submits, race conditions, optimistic locking conflicts | Data integrity maintained, no duplicate records, correct conflict resolution |
| **Network conditions** | Slow responses, timeouts, connection drops, retries, partial responses (if applicable) | Timeout errors surfaced, retry logic works, no hung states |
| **Auth states** | Logged in, logged out, expired session, different roles, token refresh, revoked permissions mid-session | Correct 401/403 responses, graceful session expiry, role-based access enforced |
| **Data variations** | Special characters (`<>&"'/\`), Unicode (emoji, CJK, RTL), very long strings (10K chars), HTML/script injection, SQL injection patterns, null bytes | Input sanitized, no injection, data round-trips correctly |
| **Navigation patterns** | Back button, direct URL access, deep linking, refresh mid-flow, browser history, bookmark stale URLs (if applicable) | Consistent state, no double-submits, correct redirects |
| **Partial operations** | Operation interrupted mid-way, partial data saved, half-completed multi-step flows, browser close mid-upload | Clean rollback or resume, no orphaned data, clear user feedback |
| **External dependency failures** | Third-party API down, rate limited, returning unexpected data, returning slow, certificate errors | Graceful degradation, meaningful error messages, no data loss |
| **Ordering & timing** | Operations arriving out of order, delayed webhooks, clock skew, operations just before/after a deadline | Correct ordering enforced, deadlines respected, no race conditions |
| **Resource exhaustion** | Disk full, memory pressure, too many connections, file descriptor limits, quota exceeded | Graceful failure, clear error messages, no crash, resources released |

### 4.4 Test Categories to Cover

Every specification MUST include tests from ALL of these categories. If a category doesn't apply, document why.

1. **Core User Journeys**: Complete end-to-end flows for each primary scenario from Round 2. These are multi-step tests that exercise the full flow from start to finish, including all side effects.
2. **Feature-Specific Tests**: Detailed tests for each functional requirement covering:
   - Happy path with concrete inputs and expected outputs
   - Every failure mode identified in the requirement's business rules
   - Side effects (what changes beyond the direct response)
3. **Side Effect Tests**: Dedicated tests that verify observable consequences:
   - "After creating an order, verify: inventory decremented, confirmation email queued, audit log entry written, analytics event published"
   - "After deleting a user, verify: user data removed, associated files deleted, sessions invalidated, linked records updated"
4. **State Machine Tests** (when applicable): For entities with lifecycle states:
   - Test every valid transition with assertions on the resulting state
   - Test every invalid transition and verify it's rejected with the correct error
   - Test state-dependent behavior (e.g., only published items are searchable)
5. **Error Handling Tests**: Invalid inputs, service failures, network errors, race conditions, data validation failures. Each test must verify: correct error code/message, no partial state corruption, appropriate logging.
6. **Security Tests**: Authentication failures, authorization violations, injection attempts, data protection validation, CSRF, token manipulation.
7. **Data Integrity Tests**: Verify that operations maintain data consistency:
   - Foreign key / reference integrity after mutations
   - No orphaned records after cascading operations
   - Correct totals/aggregates after batch operations
   - Transaction atomicity (all-or-nothing for multi-step operations)
8. **Performance Baseline Tests**: Response time expectations, load handling, resource limits (these define the contract, actual load testing is separate)
9. **Integration Tests**: Third-party service interactions, API contract validation, failure modes of external dependencies, webhook delivery verification
10. **Cross-Scenario Tests**: Interactions between scenarios identified in Round 2b. "What happens if user A is mid-checkout and user B buys the last item?"

### 4.5 Test Sufficiency Rules

Before presenting tests to the user, verify these minimum coverage rules:

- **Every FR has at least 3 tests**: one happy path, one failure, one edge case. Complex FRs need more.
- **Every scenario has at least 5 tests**: happy path, 2 failure paths, 1 edge case, 1 side-effect verification. Complex scenarios need more.
- **Every side effect mentioned in any FR or scenario has at least 1 dedicated test** that explicitly verifies it.
- **Every error message or error code mentioned in any FR has a test** that triggers it and verifies the exact message/code.
- **Every state transition (if applicable) has a test** for both the valid transition and the invalid rejection.
- **Failure tests outnumber happy-path tests.** There are always more ways things can go wrong than go right. If the happy:failure ratio is > 1:1, add more failure tests.

### 4.6 User Review of Test Plan

Before proceeding to spec generation, present the complete test list to the user:
1. Show a summary table: Test ID, Category, Scenario it validates, Priority
2. Show the count by category (e.g., "8 happy path, 14 failure, 12 edge case, 6 side effect, 4 state transition, 4 security, 3 data integrity, 2 performance")
3. Show the **happy:failure ratio** and confirm it meets the > 1:1 failure-to-happy rule
4. **When `PROJECT_MODE = existing`**: Also show counts of modified and removed tests
5. Ask: "Does this test plan cover everything you care about? Any scenarios, side effects, or edge cases missing?"
6. If the user identifies gaps, add tests and re-present
7. Only proceed to Phase 5 after the user confirms the test plan

---

## PHASE 5: SPECIFICATION GENERATION

### Step 1: Suggest Spec Title

Before writing the specification, propose a short, descriptive title for the spec file and ask the user to confirm or change it using **AskUserQuestion**:
- Derive a slug from the project name or change description (e.g., "task-management-app", "add-oauth-support")
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

**The template adapts based on PROJECT_MODE.** Sections marked `[existing only]` are only included when `PROJECT_MODE = existing`. Sections marked `[greenfield only]` are only included when `PROJECT_MODE = greenfield`.

Generate the specification file with this structure:

```markdown
# [Project Name] — Specification Document

> Generated on: [date]
> Project: [project name]
> Version: 1.0
> Status: Draft
> Type: [Greenfield Specification | Evolution Specification]

## 1. Executive Summary
[Brief description of the project, its purpose, and target audience]
[When existing: also describe what this specification adds/changes and why]

## 2. Current State Analysis [existing only]

### 2.1 Project Overview
[Summary of the current project based on codebase reading]

### 2.2 Existing Specifications
[Summary of existing specs in specs/ — list each with a one-line description]

### 2.3 Relevant Architecture
[Current architectural elements that will be affected by the new specification]

## 3. Scope
### 3.1 In Scope
[Bulleted list of what's included]
### 3.2 Out of Scope (Non-Goals)
[Bulleted list of what's explicitly excluded]

## 4. User Personas & Actors
[Description of each user type/role]

## 5. Usage Scenarios
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

## 6. Functional Requirements

[When greenfield: all requirements use FR-XXX IDs]
[When existing: use delta-aware IDs as shown below]

### New Requirements [existing only — use for all requirements when greenfield, with FR-XXX IDs]
#### FR-NEW-001: [Requirement Title]
- **Description:** [What the system must do]
- **Inputs:** [What data comes in]
- **Outputs:** [What the system produces]
- **Business Rules:** [Constraints and logic]
- **Priority:** [Must-have / Should-have / Nice-to-have]
[Repeat for each new requirement]

### Modified Requirements [existing only]
#### FR-MOD-001: [Requirement Title] (references [original spec] FR-XXX)
- **Original behavior:** [What it did before]
- **New behavior:** [What it must do now]
- **Reason for change:** [Why]
- **Business Rules:** [Updated constraints]
[Repeat for each modified requirement]

### Removed Requirements [existing only]
#### FR-DEL-001: [Requirement Title] (references [original spec] FR-XXX)
- **Description:** [What is being removed]
- **Reason:** [Why]
- **Cleanup:** [What code, data, config must be cleaned up]
[Repeat for each removed requirement]

## 7. Non-Functional Requirements
[When existing: only NFRs that are new or changed — reference existing spec NFRs where applicable]

### 7.1 Performance
[Specific, measurable requirements]
### 7.2 Security
[Authentication, authorization, data protection]
### 7.3 Usability
[UX standards, accessibility]
### 7.4 Reliability
[Uptime, recovery, backup]
### 7.5 Observability
[OpenTelemetry is mandatory. Specify:]
- **Collector**: [JSONL file (default) | OTLP to Jaeger/Tempo/etc.]
- **Trace file path**: [e.g., `traces/app.jsonl`]
- **What to trace**: API calls (INFO), DB queries (DEBUG), external tool calls (INFO), file mutations (DEBUG), auth ops (INFO), errors (ERROR), warnings (WARNING)
- **LLM tracing** (if applicable): model, token counts, duration, cost — NEVER prompts/responses
- **Custom sub-metrics**: [project-specific spans beyond defaults]
- **Sensitive data exclusion**: LLM prompts/responses, credentials, tokens, PII — NEVER in traces
[Additional logging, monitoring, alerting requirements]
### 7.6 Deployment
[Infrastructure, CI/CD, environments]
### 7.7 Scalability
[Growth and scaling strategy]

## 8. Data Model
[Key entities and their relationships — can be a table or diagram description]

## 9. Impact Analysis [existing only]

### 9.1 Affected Components
| File/Module | Impact Type | Description |
|-------------|-------------|-------------|
[Table of affected source files with what changes are needed]

### 9.2 Affected Requirements
| Spec File | Requirement ID | Impact | Description |
|-----------|----------------|--------|-------------|
[Table cross-referencing existing spec requirements that are impacted]

### 9.3 Affected Tests
| Test File | Test ID/Name | Action | Description |
|-----------|-------------|--------|-------------|
[Table of existing tests that need modification, removal, or highlight gaps]

### 9.4 Affected Documentation
| Document | Section | Action | Description |
|----------|---------|--------|-------------|
[Table of documentation that needs updating]

### 9.5 Dependencies & Risks
[New dependencies, removed dependencies, breaking changes, migration needs, rollback considerations]

## 10. Documentation Requirements

All documentation listed below MUST be created (greenfield) or updated (existing) as part of this project.

### 10.1 README.md
- Project description, purpose, and audience
- Prerequisites and installation instructions
- How to run, build, and test the project
- Configuration and environment variables
- Usage examples

### 10.2 CLAUDE.md & .agent_docs/
- `CLAUDE.md`: Compact index with project overview, key commands, essential conventions, and documentation index referencing `.agent_docs/` files
- `.agent_docs/*.md`: Detailed documentation organized by topic (architecture, API, data model, deployment, etc.)
- Must be kept in sync with code changes

### 10.3 docs/*
- User-facing documentation (guides, tutorials, API reference)
- Architecture and design decision records
- Operational runbooks (if applicable)

## 11. Traceability Matrix

| Scenario | Functional Req | E2E Tests (Happy) | E2E Tests (Failure) | E2E Tests (Edge) |
|----------|---------------|-------------------|---------------------|-------------------|
| SC-001 | FR-001, FR-002 | E2E-001 | E2E-005, E2E-006 | E2E-010 |
[Every scenario MUST have at least one test in each column. Every FR MUST appear at least once.]
[When existing: use FR-NEW/FR-MOD/FR-DEL and E2E-NEW/E2E-MOD/E2E-DEL IDs as appropriate]

## 12. End-to-End Test Suite

> **This is the most important section of the specification.** E2E tests are the primary contract for agent-based implementation. An agent will use these tests as its definition of "done": the implementation is correct when all E2E tests pass. Each test must be precise enough that an agent can write both the test code and the production code from it alone.

All tests MUST be implemented in the `tests/` directory. Each feature MUST have tests covering happy paths, failure paths, side effects, edge cases, and error recovery. **Failure tests must outnumber happy-path tests.**

### 12.1 Test Summary
| Test ID | Action | Category | Scenario | FR refs | Priority |
|---------|--------|----------|----------|---------|----------|
[Table of all tests. Action = New (for greenfield, all are New) / Modified / Removed]
[Each test must link to both a scenario (SC-XXX) and requirement(s) (FR-XXX)]

**Coverage Statistics:**
- Happy path: [N] tests
- Failure/error: [N] tests
- Side effects: [N] tests
- Edge cases: [N] tests
- State transitions: [N] tests
- Security: [N] tests
- Data integrity: [N] tests
- Performance: [N] tests
- Happy:Failure ratio: 1:[X] (must be > 1:1 failure-to-happy)

### 12.2 New Test Specifications
#### E2E-001: [Test Name] [greenfield]
#### E2E-NEW-001: [Test Name] [existing]
- **Category:** [Core Journey / Feature / Error / Side Effect / State Transition / Security / Data Integrity / Performance / Cross-Scenario]
- **Scenario:** SC-XXX — [Which usage scenario this validates]
- **Requirements:** FR-XXX, FR-YYY — [Which functional requirements this covers]
- **Preconditions:**
  - [Specific data that must exist before the test — with concrete values]
  - [System state required (e.g., "user 'alice' exists with role 'admin'")]
  - [External services/mocks required]
- **Steps:**
  - Given [initial state — with concrete test data, e.g., "a user with email 'bob@test.com' and balance 100"]
  - When [action — with exact inputs, e.g., "POST /api/orders with body {product_id: 'P1', quantity: 2}"]
  - Then [expected direct outcome — with exact values, e.g., "response status is 201 and body contains {order_id: <uuid>, status: 'pending', total: 50.00}"]
  - And [side effect verification — e.g., "inventory for product 'P1' is decremented from 10 to 8"]
  - And [additional side effect — e.g., "an order_created event is published with {order_id, user_id, total}"]
  - And [state verification — e.g., "the order appears in the user's order list with status 'pending'"]
- **Cleanup:** [What to tear down after the test, if anything]
- **Priority:** [Critical / High / Medium / Low]
[Repeat for each new test]

### 12.3 Modified Test Specifications [existing only]
#### E2E-MOD-001: [Test Name] (was [original test reference])
- **Original test:** [What it validated before]
- **Modified to validate:** [What it validates now]
- **Preconditions:** [Updated setup]
- **Steps:**
  - Given [initial state — with concrete test data]
  - When [action — with exact inputs]
  - Then [expected outcome — with exact values]
  - And [side effect verifications]
- **Cleanup:** [Teardown if needed]
[Repeat for each modified test]

### 12.4 Removed Tests [existing only]
#### E2E-DEL-001: [Test Name] (was [original test reference])
- **Reason:** [Why this test is no longer needed]
[Repeat for each removed test]

## 13. Consistency Notes [existing only]
[Any inconsistencies found with existing specs, how they were discussed with the user, and the resolution decisions made. Existing specs are NOT modified — only referenced here.]

## 14. Migration & Implementation Notes [existing only]
[Specific implementation guidance, suggested order of operations, feature flags, data migration steps, rollback strategy]

## 15. Open Questions & TBDs
[Items that need further clarification]

## 16. Glossary
[Domain-specific terms and definitions]
```

### Step 5: Commit and summary

1. Commit the spec file: `git add specs/ && git commit -m "Add specification: <title>"`
2. If deferred ideas exist, also commit `specs/BACKLOG.md`
3. Provide a brief summary of what was generated:
   - **Greenfield**: counts of requirements, tests, scenarios, documentation requirements, and deferred ideas sent to backlog
   - **Existing**: counts of new/modified/removed requirements, new/modified/removed tests, affected components, and deferred ideas sent to backlog
4. Tell the user they can use `/project:push` when ready to push and create a pull request.

---

## QUALITY ASSURANCE CHECKLIST

Before delivering the final specification, verify:

### Interview & Process
- [ ] Project state was detected (greenfield vs existing)
- [ ] When existing: project was thoroughly read (Phase 0) before interview
- [ ] 2-3 approaches were proposed and user chose one before spec generation
- [ ] YAGNI was applied — no unnecessary features included; deferred ideas written to `specs/BACKLOG.md`
- [ ] Every usage scenario was confirmed with the user in structured format (Actor/Preconditions/Flow/Postconditions/Exceptions)
- [ ] Failure/error scenarios were explicitly probed for each scenario (Round 2c)
- [ ] Test plan was reviewed and approved by the user before spec generation (Phase 4.5)

### Impact & Consistency (existing projects only)
- [ ] Impact analysis covers all affected components, requirements, tests, and documentation
- [ ] Every inconsistency with existing specs was flagged and resolution documented
- [ ] Existing specs are NOT modified — only referenced
- [ ] Every modified feature has updated E2E tests
- [ ] Every removed feature has its tests marked for removal

### E2E Test Quality (this is the most important section)
- [ ] Every FR has at least 3 tests (happy, failure, edge case); complex FRs have more
- [ ] Every scenario has at least 5 tests (happy, 2 failure, edge case, side effect)
- [ ] Failure tests outnumber happy-path tests (happy:failure ratio < 1:1)
- [ ] Every side effect mentioned in any FR or scenario has a dedicated verification test
- [ ] Every error message/code mentioned in any FR has a test that triggers and verifies it
- [ ] Every state transition (if applicable) has both valid and invalid transition tests
- [ ] Every "Then" assertion specifies exact values, not vague outcomes
- [ ] Every test includes concrete test data (not "valid input" but actual values)
- [ ] Every test specifies the verification method (API check, DB query, event check, etc.)
- [ ] Coverage statistics are included in the test summary

### Traceability (the matrix must be complete)
- [ ] Every scenario (SC-XXX) has at least one E2E test for happy path, failure path, AND edge cases
- [ ] Every functional requirement (FR-XXX) is covered by at least one E2E test
- [ ] Every E2E test references both its scenario (SC-XXX) and its requirements (FR-XXX)
- [ ] No orphan tests (tests that don't link to a scenario or requirement)
- [ ] No orphan requirements (FRs with no test coverage)
- [ ] No orphan scenarios (scenarios with no E2E test)

### Spec Quality
- [ ] Every non-functional requirement has measurable acceptance criteria
- [ ] No ambiguous language (avoid "should", "might", "could" — use "must", "will")
- [ ] All requirements have unique IDs (FR-XXX for greenfield; FR-NEW/FR-MOD/FR-DEL-XXX for existing)
- [ ] All scenarios have unique IDs (SC-XXX)
- [ ] All tests have unique IDs (E2E-XXX for greenfield; E2E-NEW/E2E-MOD/E2E-DEL-XXX for existing)
- [ ] Documentation requirements (README.md, CLAUDE.md, .agent_docs/, docs/*) are specified
- [ ] Section depth is proportional to complexity — no padded sections
- [ ] Open questions are clearly flagged as TBDs
- [ ] The document is self-contained and understandable without additional context
- [ ] The spec file is written to `specs/<timestamp>-<title>.md`

---

## BEHAVIORAL RULES

1. **Always analyze the project first.** Phase 0 is mandatory — detect greenfield vs existing and read accordingly.
2. **Never skip the interview phase.** Even if the user provides upfront context, validate and probe deeper.
3. **Always use AskUserQuestion tool** for the interview — do not make assumptions.
4. **Prefer multiple choice questions** — use AskUserQuestion options whenever possible to make the interview faster and more focused.
5. **Be methodical** — follow the phases in order: context → interview → approaches → impact (if existing) → tests → spec.
6. **Be precise** — specifications must be unambiguous and testable.
7. **Be complete** — cover happy paths, failure modes, AND edge cases.
8. **Be adaptive** — adjust interview depth based on project mode. Existing projects need fewer questions focused on what's changing.
9. **Apply YAGNI ruthlessly** — actively challenge features that aren't essential for MVP. If the user agrees to defer an idea, write it to `specs/BACKLOG.md` (create the file if it doesn't exist, append if it does). Each backlog entry must have: ID, idea title, brief description, rationale for deferral, and which spec suggested it.
10. **Always propose 2-3 approaches** — never jump straight from interview to specification. Present alternatives with trade-offs and your recommendation.
11. **Scale sections to complexity** — adapt the depth of each spec section to its actual complexity. Don't pad simple sections.
12. **Present a summary of findings** after the interview and before generating tests/specs, asking for final confirmation.
13. **Number everything** — all requirements (FR-XXX), scenarios (SC-XXX), and tests (E2E-XXX) must have traceable IDs. Use delta-aware IDs (FR-NEW/FR-MOD/FR-DEL) for existing projects.
14. **Write the spec file** to `specs/<YYYY-mm-dd_HH:MM:SS>-<title>.md` — never to the project root, never overwrite existing specs.
15. **Suggest a spec title** and ask the user to confirm before writing.
16. **After writing the specification**, provide a brief summary of what was generated.
17. **Ask about existing project context** if project files exist — never ignore them without asking.
18. **Confirm each scenario individually** — after deep-diving into a scenario, present it in structured format and get user confirmation before moving to the next.
19. **Probe failures explicitly** — never accept "nothing can go wrong" without pushing back. Dedicate Round 2c entirely to failure and error scenarios.
20. **Get user sign-off on the test plan** — present the full test list (Phase 4.5) and only proceed to spec generation after the user approves it.
21. **Build the traceability matrix** — every scenario must link to FRs and E2E tests. No orphans allowed.
22. **Never modify existing specs.** Only reference them. Flag inconsistencies and document resolutions in the new spec.
23. **Be thorough on impact** (existing projects) — the value of an evolution spec is in the impact analysis. Miss nothing.
24. **Flag all inconsistencies** with existing specs during the interview — do not silently ignore them.
25. **E2E tests are the primary deliverable.** The spec is the context; the E2E tests are the contract. An agent implementing code uses E2E tests as its definition of "done". Invest maximum effort in test completeness, precision, and coverage.
26. **Never write vague test assertions.** Every "Then" must specify exact expected values, status codes, response bodies, or observable state changes. "The system handles the error" is not an assertion; "response status is 400 with body {error: 'email_required'}" is.
27. **Always verify side effects.** Every operation that creates, modifies, or deletes data must have tests that verify not just the direct response but all observable consequences (database changes, events published, notifications sent, caches invalidated, related entities updated).
28. **Failure tests must outnumber happy-path tests.** There are always more ways for things to go wrong than go right. If the ratio is off, actively generate more failure and edge case tests before presenting to the user.
29. **Use concrete test data.** Never use placeholders like "valid input" or "some user". Specify exact values: `{"name": "Alice", "email": "alice@test.com"}`. This removes ambiguity for the implementing agent.
