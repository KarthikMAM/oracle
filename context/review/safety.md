# safety

Could this hurt at runtime? Concurrency, async hygiene, untrusted input, defensive coding, security, crypto, data integrity.

## Why

Safety defects are the most expensive category: data loss, security breaches, cascading failures. Unlike correctness bugs (wrong output), safety bugs produce catastrophic output — crashes, deadlocks, data corruption, security vulnerabilities.

## Attack Patterns

1. **trust-boundary-trace**: Identify each trust boundary the diff crosses. Trace data flow from boundary to use site. Verify validation occurs before use.
2. **shared-state-map**: List all shared mutable state touched by the diff. Identify the synchronization mechanism for each. One-and-only-one mechanism per state. Flag unprotected or multiply-protected state.
3. **lock-vs-await-scan**: For each lock/mutex acquisition, scan to its release. If an `await`/`suspend` lies between acquisition and release → deadlock window.
4. **closure-capture-audit**: For each escaping closure/lambda, list captured `self`/`this` references. Long-lived holder + strong capture → retain cycle.
5. **secret-scan**: Regex for tokens, API keys, hardcoded production URLs, HTTP (non-HTTPS) endpoints, credentials.
6. **crypto-enumeration**: List every cryptographic call. Reject MD5/SHA1-for-security/DES/RC4. Reject hand-rolled primitives.
7. **fire-and-forget-detection**: Async work without cancellation handling. Every `Task`/`Promise`/coroutine MUST be tied to a lifecycle or cancellation token.
8. **resource-leak-on-error-path**: For each resource opened, verify cleanup on ALL exit paths — not just happy path. `finally`/`defer`/`use` required.
9. **injection-vector-scan**: Scan for string concatenation building SQL, HTML, URLs, or shell commands with user-supplied data.
10. **timeout-completeness**: Every external async operation (network, IPC, file I/O) MUST have a documented timeout.

## Severity

- **BLOCK**: Concrete repro for data loss, security breach, deadlock, retain cycle, or silent corruption.
- **SHOULD**: Plausible failure under unusual but realistic load/input; mitigation is cheap.
- **NIT**: Hardening that strengthens defense-in-depth without a known failure case.

## Finding Format

BLOCK findings MUST include:
- Concrete repro path (input X → failure Y)
- Steel-man counter-argument (why this might be false positive)
- Counter-argument resolution

Templates:
- "Race between guard check at {file:line} and dispatch at {file:line} — state can change between."
- "Missing timeout on external HTTP call at {file:line} — could hang indefinitely."
- "User input at {file:line} flows to SQL string concatenation at {file:line} — injection risk."
- "Closure at {file:line} captures `self` strongly; the holder is long-lived → retain cycle."
- "MD5 used for token derivation at {file:line} — reject; use HKDF-SHA256."
- "Lock acquired at {file:line} held across `await` at {file:line} — deadlock window."

## Critical Rules

- BLOCK findings MUST include a concrete repro path.
- "Defense-in-depth" without a known failure case is NIT, not BLOCK.
- Hand-rolled crypto is BLOCK regardless of apparent correctness.
- Every finding MUST cite file:line evidence.
