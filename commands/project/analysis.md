Act as a Principal Software Engineer. Analyze the following report thoroughly. Load the appropriate skills to ensure the proposed solution is compliant with project standards.

First, read the project's CLAUDE.md and any relevant source files to understand the codebase context before answering.

Provide the following structured analysis:

## A. Situation Assessment
Summarize the reported issue in your own words. Classify severity (Critical / High / Medium / Low) and identify the affected domain (e.g., API, data pipeline, frontend, infrastructure).

## B. Root Cause Analysis (RCA)
Why did this occur at the code and architectural level? Identify the exact component, function, or configuration at fault. Distinguish between the root cause and contributing factors.

## C. Impact Assessment
1. **User impact:** How are end users or operations affected? (scope, frequency, workaround availability)
2. **Component impact:** What other components, services, or data flows are affected or at risk?

## D. Steps to Reproduce
Provide a clear, numbered sequence to reproduce the issue. Include preconditions, input data, and expected vs actual behavior.

## E. Solution Plan
1. **Immediate fix:** The minimal change to resolve the issue.
2. **Structural fix (if needed):** Any refactoring or architectural change to prevent recurrence.
3. **Tests to add:** Specific end to end and non regression tests covering this scenario and edge cases.
4. **Rollback plan:** How to revert safely if the fix introduces regressions.

## F. Verification Plan
Define the exact acceptance criteria: which commands to run, which tests must pass, and which manual checks to perform to confirm the fix is complete. If verification fails, iterate on the solution until all criteria are met.

<report>
$ARGUMENTS
</report>

