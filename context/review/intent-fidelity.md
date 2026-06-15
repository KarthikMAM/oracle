# intent-fidelity

Does the artifact honor user constraints, directive compliance, and requirement coverage? Specifically catches deviations between the user's stated intent and the implementation.

## Why

The most expensive failure mode in autonomous coding is "code that works but solves the wrong problem." This lens specifically checks that what was BUILT matches what was REQUESTED — not just that the build is green.

## Attack Patterns

1. **directive-compliance**: Re-read user's verbatim request from intake.md. Every explicit instruction MUST be honored or surfaced as a deviation requiring user decision. Flag silent overrides.
2. **scope-creep-detection**: Compare diff scope against intake.md `In Scope` and `Out of Scope` sections. Flag any file modified outside declared scope. Flag any feature added not requested.
3. **requirement-coverage**: Walk every requirement in requirements.md. Verify implementation addresses it. Flag silent omissions.
4. **deviation-explicitness**: design.md MUST list every deviation from user's stated standard or cited spec in "Deviations Requiring User Decision" section. Flag intentional deviations not surfaced.
5. **constraint-honor**: Walk constraints in requirements.md. Verify implementation respects them. Flag violations.
6. **acceptance-criteria-coverage**: Every acceptance criterion (Given/When/Then) MUST have a corresponding test. Flag missing tests.
7. **assumption-verification**: Walk assumptions in intake.md. Verify implementation matches the assumption (or surfaces unverified assumption).
8. **unstated-requirement-leakage**: Flag implementations that depend on requirements NOT stated in intake/requirements (hidden assumptions baked into code).

## Severity

- **BLOCK**: Silent override of user's explicit instruction. Implementation outside declared scope. Missing requirement coverage.
- **SHOULD**: Unsurfaced deviation. Missing acceptance test. Unverified assumption baked in.
- **NIT**: Minor scope expansion that's clearly beneficial but should be confirmed.

## Finding Format

Templates:
- "User's intake.md §3 says 'no breaking API changes' — diff at {file:line} removes public method `getUserId()`. BLOCK; surface as deviation or restore."
- "requirements.md R5 mandates retry on 503 errors — implementation at {file:line} does not retry. BLOCK."
- "design.md states alternative B was rejected — diff at {file:line} implements alternative B partially. BLOCK; surface to user."
- "intake.md §scope says 'do not modify auth flow' — diff modifies src/auth/Login.ts. BLOCK; revert or surface."

## Critical Rules

- Findings MUST cite the specific intake/requirements line that is violated.
- "Could be wrong" without a demonstrated drift is SHOULD at most.
- A deviation explicitly approved at user checkpoint is NOT a violation.
- The user's verbatim words in intake.md are ground truth — paraphrases or interpretations are not.
