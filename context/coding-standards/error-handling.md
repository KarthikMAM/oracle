# Error Handling

A failure that crosses a boundary untyped, gets swallowed, or fails open is invisible at the time it is created and surfaces later as a corrupted state, a stuck request, or a 3 a.m. page; the boundary is the one place where a raw failure becomes a value the caller can reason about. **Posture (per `context/principles.md` → Simplicity):** errors are values, not hidden control flow — surface them explicitly, handle them at the call site, and keep the failure path as visible as the happy path. This is the Go ethos applied in each language's own idiom (a `Result`/`safeRun` boundary, a checked return, a typed throw caught at the edge — never a swallowed exception or a silent fail-open).

## Anchors

- **Errors Are Values** (Pike, 2015) — failures are ordinary return values to be inspected, not invisible control flow that teleports up the stack
- **Catch at the Boundary** (Bloch — *Effective Java* item 73, 2018) — translate low-level exceptions into the abstraction's own error type at the layer boundary; never let an implementation detail leak upward
- **Don't Swallow Exceptions** (Bloch — *Effective Java* item 77, 2018) — an empty `catch` defeats the entire purpose of exceptions and hides the bug that caused them
- **Parse, Don't Validate** (Alexis King, 2019) — convert a failure into a typed value once, at the edge, so the rest of the system handles a known shape
- **Fail-Safe Defaults** (Saltzer & Schroeder, 1975) — when a check cannot complete, deny; an error must never be silently treated as success
- **The Tail at Scale** (Dean & Barroso, 2013) — without bounded, jittered retries on a retriable-only predicate, one failing node amplifies into a fleet-wide retry storm
- **Stability Patterns** (Nygard — *Release It!*, 2nd ed. 2018) — circuit breakers, timeouts, and bulkheads stop a single failing dependency from cascading; a retry without a breaker is a self-inflicted denial of service

## Q7 — General Rules

Untyped exceptions MUST NOT cross a module boundary; each boundary converts failures into the language's idiomatic error-as-value representation. Violations are BLOCK-severity for the safety reviewer lens.

- **MUST** — every trust boundary (bridge, network, IPC, user input, deserialization, file I/O, database, third-party SDK call, subprocess, environment/config read, clock/random source) convert failures into the boundary's error-as-value representation; rejected Promises, unobserved futures, unhandled error channels, and uncaught callback errors never escape unhandled.
- **MUST** — on a recoverable failure, return a safe documented default OR propagate a typed error; "recoverable" is a deliberate, documented determination, never reached solely because handling is inconvenient.
- **MUST** — map low-level errors to a domain-specific type at the boundary. **MUST NOT** — rethrow them unchanged past it.
- **MUST NOT** — catch in the middle of a call chain; catch at the boundary or let the failure propagate to one. **MUST NOT** — a catch whose only act is to log-and-rethrow (Q7.4).
- **MUST NOT** — catch generic `Exception` / `Error` / `Throwable` / `catch (e)` / `catch (...)` / bare `except:` except inside the sanctioned `safeRun` body (Q7.1) at the outermost boundary. **SHOULD** — name the narrowest error type the boundary can actually produce. *Escape hatch: the broad catch inside `safeRun` itself, and a foreign callback whose contract can throw an open-ended set, are exempt.*
- **MUST** — retry logic lives in a dedicated retry wrapper with a finite ceiling and a retriable-error predicate; **SHOULD** — back off exponentially with jitter (Resource Management Q12.5, which owns the strategy). *Escape hatch on the strategy: a low fixed-ceiling backoff, a decorrelated-jitter scheme, or deferring to a circuit breaker is acceptable when the call site documents why; the ceiling and predicate stay mandatory (Q12.5).* **MUST NOT** — inline ad-hoc retries in business logic. **MUST** — retries only wrap idempotent operations or operations guarded by an idempotency key (Q7.12).
- **MUST** — a boundary that retries or fans out to a dependency is protected by a circuit breaker, bulkhead, or load-shed (Q7.13).
- **MUST NOT** — silently swallow a failure (Q7.5), use it as ordinary control flow (Q7.6), default it to "success"/"permitted" (Q7.7), or report it to the user as success (Q7.14).
- **MUST** — model a timeout, deadline, or cancellation as a first-class typed error, not a hang, a `null`, or an indistinguishable generic failure (Q7.11).

## Q7.1 — The safeRun Boundary Mechanism

The sanctioned boundary mechanism is ONE project util — `safeRun` / `runSafe` — that performs the try/catch internally and returns a structured `SafeResult` carrying success/value/error. Its body is the one acceptable raw try/catch site; concentrating the raw catch in one audited function is what lets the linter ban try/catch everywhere else.

- **MUST NOT** — business-logic functions contain a raw try/catch; they call `safeRun` at the boundary and branch on the returned `SafeResult`.
- **MUST** — callers inspect `success` / `error` and act on both arms (safe documented default or typed error propagation). **MUST NOT** — discard a `SafeResult` or let a reachable failure arm compile away to a no-op; both arms are exhaustively matched.
- **MUST** — async trust boundaries use `safeRunAsync`, which `await`s the block inside the try. **MUST NOT** — a synchronous `safeRun` wrapping an un-awaited Promise/future/lazy sequence (it captures nothing; the rejection escapes).
- **MUST NOT** — ad-hoc inline `runCatching` / `runSafeCatching` / `try?` / `try!` / `?? <default>`-over-a-throw scattered through business logic. The objection is the *scattering*, not the stdlib type: `safeRun` MAY itself be a thin wrapper that returns or adapts `kotlin.Result` (or Swift `Result`) — it just has to be the single audited site that encodes the Q7.8 carve-outs (re-throw cancellation, never catch fatal), which the bare stdlib forms do not.
- **MUST NOT** — nest a `safeRun` inside another `safeRun` to "double-handle" the same failure; the inner failure is already a value — branch on it.
- *Escape hatch: a single additional raw try/catch site is sanctioned ONLY where a foreign API mandates it and `safeRun` cannot wrap it — a `finally`-only cleanup with no value, a framework callback whose signature forbids a wrapper, or a hot loop where the wrapper's allocation is **measured** to matter. Such a site MUST carry an inline comment naming the constraint, MUST still map the error to a domain type, and MUST still honor the cancellation/fatal carve-outs of Q7.8. Convenience is not a constraint.*

The PRINCIPLE is universal; the IMPLEMENTATION varies by language idiom. The util itself MUST encode the Q7.8 carve-outs (re-throw cancellation, never catch fatal) so every boundary inherits them for free.

**TypeScript:**
```typescript
type SafeResult<T> =
  | { success: true; value: T }
  | { success: false; error: Error }

function safeRun<T>(fn: () => T): SafeResult<T> {
  try {
    return { success: true, value: fn() }
  } catch (e) {
    return { success: false, error: e instanceof Error ? e : new Error(String(e)) }
  }
}

async function safeRunAsync<T>(fn: () => Promise<T>): Promise<SafeResult<T>> {
  try {
    return { success: true, value: await fn() }
  } catch (e) {
    return { success: false, error: e instanceof Error ? e : new Error(String(e)) }
  }
}
```

**Kotlin:**
```kotlin
sealed class SafeResult<out T> {
  data class Success<T>(val value: T) : SafeResult<T>()
  data class Failure(val error: Throwable) : SafeResult<Nothing>()
}

inline fun <T> safeRun(block: () -> T): SafeResult<T> =
  try {
    SafeResult.Success(block())
  } catch (e: CancellationException) {
    throw e                       // never swallow cancellation (Q7.8)
  } catch (e: Throwable) {
    SafeResult.Failure(e)
  }
```

**Swift:** mirror the Kotlin shape with `Result<T, Error>`; in the async form `catch is CancellationError` first to surface (not mask) cancellation (Q7.8) before the general `catch`.

**Python:**
```python
def safe_run(fn: Callable[[], T]) -> Tuple[Optional[T], Optional[Exception]]:
    try:
        return (fn(), None)
    except Exception as e:           # NOT bare `except:` — never catch SystemExit/KeyboardInterrupt
        return (None, e)

async def safe_run_async(fn: Callable[[], Awaitable[T]]) -> Tuple[Optional[T], Optional[Exception]]:
    try:
        return (await fn(), None)
    except asyncio.CancelledError:   # re-raise cancellation (Q7.8)
        raise
    except Exception as e:
        return (None, e)
```

**Go:** already idiomatic — `(val, err)` is the convention; no wrapper needed. **MUST** — every returned `err` is checked. **MUST NOT** — a discarded error (`val, _ := ...`) on a fallible call unless the discard carries a comment justifying why the failure is genuinely irrelevant. **MUST** — a recovered `panic` is converted to an `error` and returned, never discarded.

**Rust:** native `Result<T, E>` with `?` IS the principle; the compiler enforces handling. **MUST NOT** — `.unwrap()` / `.expect()` / `panic!` / `unreachable!` / array-index `[i]` on attacker-influenced bounds on a fallible boundary call in production paths. **MUST NOT** — `let _ = fallible()` that drops a `Result` without a justifying comment.

```kotlin
// RIGHT — boundary call with exhaustive match
when (val result = safeRun { repository.fetchUser(id) }) {
    is SafeResult.Success -> render(result.value)
    is SafeResult.Failure -> {
        logger.error("fetchUser failed", mapOf("userId" to id), result.error)
        renderError("Could not load user")
    }
}

// WRONG — inline stdlib runCatching scattered in business logic, and `.getOrNull()` swallows
// the failure (and does not re-throw CancellationException). The defect is the inline scatter +
// silent drop, not `kotlin.Result` itself — a `safeRun` that wraps `Result` and adds the Q7.8
// carve-outs is fine; calling `runCatching { … }.getOrNull()` at the call site is not.
val user = kotlin.runCatching { repository.fetchUser(id) }.getOrNull()
```

## Q7.2 — Map to Domain Error Types at the Boundary

- **MUST** — each boundary translates the failures it can produce into a closed set of documented domain error types; callers branch on the domain type, never on the underlying library's type.
- **MUST** — the domain error carries enough structure to drive the caller's decision (at minimum a stable code/category and whether the failure is retriable). **MUST NOT** — encode retriability or any machine-consumed signal in a free-text message that callers parse.
- **MUST** — the mapping is exhaustive over producible failures; an unrecognized low-level error maps to an explicit `unknown`/`unexpected` variant (non-retriable, fail-closed per Q7.7), never a default masquerading as success or as retriable.
- **SHOULD** — model the recoverability distinction the caller actually needs (`NotFound` vs `Unavailable` vs `Invalid` vs `Conflict` vs `RateLimited`). *Escape hatch: a boundary with exactly one caller and one failure mode MAY use a single error type; split it the moment a second caller needs a different reaction.*

```typescript
// WRONG — the HTTP layer's own error leaks; callers must know it's fetch under the hood
async function getUser(id: string): Promise<User> {
  const res = await fetch(`/users/${id}`)
  return res.json()            // throws raw SyntaxError / TypeError across the boundary
}

// RIGHT — closed domain error set; retriability is structured, not parsed from a string;
//         an unrecognized status maps to an explicit non-retriable `unexpected` variant.
// The returned errors ARE the declared union (no separate undeclared classes), so retriability
// is a typed field the caller branches on, and the body decode is wrapped at the boundary too.
type UserError =
  | { kind: "not_found"; retriable: false }
  | { kind: "unavailable"; retriable: true }
  | { kind: "invalid_response"; retriable: false }
  | { kind: "unexpected"; retriable: false }   // exhaustive catch-all, fails closed

type UserResult =
  | { success: true; value: User }
  | { success: false; error: UserError }

async function fetchUser(id: string): Promise<UserResult> {
  const r = await safeRunAsync(() => fetch(`/users/${id}`))
  if (!r.success) return { success: false, error: { kind: "unavailable", retriable: true } }
  if (r.value.status === 404) return { success: false, error: { kind: "not_found", retriable: false } }
  if (r.value.status >= 500) return { success: false, error: { kind: "unavailable", retriable: true } }
  if (r.value.status !== 200) return { success: false, error: { kind: "unexpected", retriable: false } }

  // Response.json() rejects with SyntaxError on a malformed body — that rejection is a trust-boundary
  // failure too, so it goes through safeRunAsync, never escapes raw (Q7, Q7.1 checklist #8).
  const body = await safeRunAsync(() => r.value.json())
  if (!body.success) return { success: false, error: { kind: "invalid_response", retriable: false } }

  const parsed = UserSchema.safeParse(body.value)
  return parsed.success
    ? { success: true, value: parsed.data }
    : { success: false, error: { kind: "invalid_response", retriable: false } }
}
```

## Q7.3 — Preserve the Cause Chain

- **MUST** — when wrapping a low-level error, attach the original as the cause (Java/Kotlin `cause`, JS `{ cause }`, Python `raise ... from e`, Swift `underlyingError`, Go `%w`, Rust `#[source]`). **MUST NOT** — a wrap that discards the cause.
- **MUST NOT** — lose the stack trace / backtrace on re-throw; `throw new Error("failed")` never replaces `throw new Error("failed", { cause: e })`.
- **MUST NOT** — collapse the cause chain to a string and re-wrap (Go `%v`/`%s`, Python `raise X(str(e))`, JS `new Error(String(e))` on an already-`Error`) where structured unwrapping (`errors.Is`/`errors.As`, `__cause__`, `instanceof`) is used — stringifying severs programmatic matching.
- **SHOULD** — the wrap message adds context the cause lacks (which entity, operation, identifier). *Escape hatch: if the cause's message is already fully self-describing, a wrap that only adds the domain type is acceptable.*

```python
# WRONG — original traceback is gone; on-call sees "load failed" and nothing else
try:
    return parse(raw)
except ValueError:
    raise ConfigError("load failed")        # cause dropped

# RIGHT — `from e` chains the cause; the original traceback is preserved
try:
    return parse(raw)
except ValueError as e:
    raise ConfigError(f"invalid config for tenant {tenant_id}") from e
```

## Q7.4 — Structured Logging at the Boundary, Once

- **MUST** — the handling boundary logs the failure once, with structured context: operation name, correlation/request id, domain error code/type, chained cause. **MUST NOT** — a re-throw also logs (log at the point of handling, not every frame); **MUST NOT** — log-and-rethrow (Q7).
- **SHOULD** — log fields are structured key/value pairs, not interpolated into free text. *Escape hatch: a single human-readable summary line alongside the structured fields is fine.*
- **MUST** — a caught error that is neither rethrown nor recovered is logged at error/warn; logging it at `debug`/`trace`/`info` is equivalent to swallowing it (Q7.5).
- **MUST** — severity matches consequence: a failure that drops data, breaches an invariant, or pages someone is `error`; a tolerated best-effort failure is `warn`. **MUST NOT** — log a genuine fault at `warn` to dodge an alert.
- **SHOULD** — attach the correlation id from ambient request context rather than threading it through every signature. *Escape hatch: in code with no request context (CLI, one-shot migration) the correlation id MAY be omitted; operation name and cause stay mandatory.*

```typescript
// WRONG — logged at every layer, unstructured, secret leaked into the message
logger.error(`failed for ${user.email} token=${authToken}: ${err}`)   // re-logged upstream too

// RIGHT — logged once at the handling boundary, structured, no secrets (see Q7.9)
logger.error("payment.capture failed", {
  op: "payment.capture",
  correlationId: ctx.correlationId,
  errorCode: domainError.code,
  userId: user.id,            // stable id, not PII
  cause: err,                 // logger serializes the chained cause + stack
})
```

## Q7.5 — Never Swallow a Failure

- **MUST** — a caught error yields at least one of: a recovery action, a propagation, or an error/warn-level log with the cause. **MUST NOT** — an empty `catch {}` / `catch (e) {}` / `except: pass` / `.catch(() => {})` / `.catch(noop)`.
- **MUST NOT** — discard the error while keeping the catch (`catch (_)` unused, `getOrNull()` / `try?` / `.ok()` / `?? default` that drops the failure) unless the swallow is deliberate; a deliberate swallow MUST carry an inline comment stating why the failure is safe to ignore AND MUST NOT sit on a path that gates access, money, or a mutation (those fail closed, Q7.7).
- **MUST NOT** — swallow in `finally` so it masks the in-flight exception; a `return`/`throw` in `finally`, or a cleanup error overwriting the original, never replaces the primary error — the primary propagates and the cleanup error is added as a suppressed/secondary cause.
- **MAY** — "best-effort" side effects (fire-and-forget metrics, cache warming) tolerate failure, but **MUST** still log at warn so a 100%-failure regression is visible AND **MUST** be marked best-effort at the call site. *Escape hatch: a tight hot best-effort path (per-frame telemetry) MAY downgrade to a sampled or rate-limited warn, never to silence.*

```kotlin
// WRONG — the parse failure vanishes; the field is silently null forever
val expiry = try { Instant.parse(raw) } catch (e: Exception) { null }

// RIGHT — deliberate fallback, but the failure is recorded so a spike is visible
val expiry = when (val r = safeRun { Instant.parse(raw) }) {
    is SafeResult.Success -> r.value
    is SafeResult.Failure -> {
        logger.warn("expiry parse failed, defaulting to never", mapOf("raw" to raw), r.error)
        Instant.MAX            // documented safe default
    }
}
```

```java
// WRONG — the finally's own throw swallows the real failure; on-call sees "close failed"
try {
    return process(stream);
} finally {
    stream.close();           // if this throws, the original exception is lost
}

// RIGHT — try-with-resources records the close failure as suppressed; primary error wins
try (stream) {
    return process(stream);   // close() failure is attached via getSuppressed(), not masked
}
```

## Q7.6 — Errors Are Not Control Flow

- **MUST** — model an expected, non-exceptional outcome ("not found", "end of input", "validation failed") as a value (`Optional`/`null`, sentinel, `Result`, sum type), NOT a thrown exception that normal flow catches.
- **MUST NOT** — use a `catch` to implement branching a conditional would express, or to break out of loops / return early.
- **MUST NOT** — use a sentinel return that overloads a valid result (`-1`, `0`, `""`, empty collection, `NaN`, `null` where those are also legitimate successes); use a distinguishable type (`Result`, `Optional` of a non-nullable, a tagged union).
- **SHOULD** — validation that produces user-facing field errors accumulates them into a returned structure rather than throwing on the first failure (Q7.10). *Escape hatch: where a library's only API throws (a parser with no `tryParse`), wrap it in `safeRun` at the call site to turn the throw back into a value immediately.*

```swift
// WRONG — "not found" is an ordinary outcome, modeled as a thrown error and caught for flow
func find(_ id: ID) throws -> Item { /* throws NotFound */ }
do { return try find(id) } catch { return .placeholder }   // exception-as-branch

// RIGHT — absence is a value; the optional makes the empty case explicit and fast
func find(_ id: ID) -> Item? { store[id] }
return find(id) ?? .placeholder
```

```go
// WRONG — -1 is also a plausible offset; callers cannot tell "not found" from a real index
func indexOf(s string, b byte) int { /* returns -1 on absent */ }

// RIGHT — the boolean second value distinguishes absence from a valid zero index
func indexOf(s string, b byte) (int, bool) {
    for i := 0; i < len(s); i++ {
        if s[i] == b {
            return i, true
        }
    }
    return 0, false
}
```

## Q7.7 — Fail Closed, Never Fail Open

- **MUST** — on error, timeout, or ambiguity in any check that gates access, authorization, a mutation, a quota, a security-meaningful feature flag, or correctness, fail closed: deny, reject, return the safe restrictive default. **MUST NOT** — a `catch` that returns `true`/`allowed`/`valid`/`admin`/the unfiltered input. (Reinforces Data Integrity Q13.6.)
- **MUST** — a default substituted on error is the SAFE default, not the convenient one ("empty allowlist" not "allow-all"; "zero balance available" not "last known balance"; "most restrictive role" not "cached role").
- **MUST NOT** — treat a partial result from a security/correctness check as a full pass; if a multi-step check cannot complete every step, the whole check fails closed.
- **MUST** — the fail-closed path is covered by a test that injects the dependency failure and asserts the denial (it is the branch that never executes during a passing run).

```typescript
// WRONG — fraud service down => everyone passes; an outage opens the gate
async function canTransfer(userId: string): Promise<boolean> {
  const r = await safeRunAsync(() => fraudService.isClear(userId))
  return r.success ? r.value : true         // FAIL-OPEN
}

// RIGHT — error denies; the invariant holds even when the dependency is down
async function canTransfer(userId: string): Promise<boolean> {
  const r = await safeRunAsync(() => fraudService.isClear(userId))
  if (r.success) return r.value
  logger.error("fraud check unavailable, denying transfer", { userId, cause: r.error })
  return false                              // FAIL-CLOSED (path is unit-tested)
}
```

## Q7.8 — Never Swallow Cancellation or Fatal Signals

- **MUST** — a broad handler re-throws cancellation signals: Kotlin `CancellationException`, Swift `CancellationError`, Python `asyncio.CancelledError`, Java `InterruptedException` (and restore the interrupt flag), Go context cancellation via `ctx.Err()`. **MUST NOT** — swallow cancellation. (Reinforces Resource Management Q12.6.)
- **MUST NOT** — a boundary handler catches fatal, non-recoverable errors — JVM `Error` subclasses (`OutOfMemoryError`, `StackOverflowError`, `LinkageError`), Python `SystemExit`/`KeyboardInterrupt` (hence `except Exception`, never bare `except:`), Rust aborts; let them terminate the process.
- **MUST NOT** — a retry wrapper retries on a cancellation or fatal signal; the retriable predicate excludes them (Q7.12, Resource Management Q12.5).
- **MUST** — the `safeRun` util encodes these carve-outs (Q7.1) so every boundary inherits them for free.

```kotlin
// WRONG — catches CancellationException too; the coroutine can never be cancelled
try {
    return fetch()
} catch (e: Throwable) {            // swallows cancellation
    return fallback
}

// RIGHT — re-throw cancellation, handle the rest. This is the ONE sanctioned raw-catch site:
// it lives inside the `safeRun`/boundary util (Q7.1), the single audited place a broad catch
// is allowed — business logic calls `safeRun` and branches on the SafeResult, it does not
// hand-roll this. The util encodes the carve-out so every boundary inherits it for free.
inline fun <T> safeRun(block: () -> T): SafeResult<T> =
    try {
        SafeResult.Success(block())
    } catch (e: CancellationException) {
        throw e                            // never swallow cancellation
    } catch (e: Throwable) {
        SafeResult.Failure(e)              // mapped to a domain type at the call site
    }
```

```java
// RIGHT — restore the interrupt flag so the thread's cancellation is not lost
try {
    return queue.poll(timeout, MILLISECONDS);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();   // re-assert interruption, then bail
    throw new ShutdownInProgress(e);
}
```

## Q7.9 — No Secrets, PII, or Raw Input in Error Surfaces

- **MUST NOT** — error messages, log fields, and exception payloads contain secrets (passwords, tokens, API keys, session ids, full card numbers, private keys) or unredacted PII beyond a stable opaque identifier; **MUST NOT** — interpolate credentials or raw request bodies into an error, or log an entire request/response/exception object that may transitively contain secrets — log named, vetted fields only.
- **MUST NOT** — an error returned across a remote/public boundary leaks internal structure (stack traces, SQL, internal hostnames, file paths, dependency names, library versions) to an untrusted caller; map to a generic client-safe message plus an opaque correlation id, log the detail internally.
- **MUST NOT** — an error varies observably (timing, message text, status code) in a way that lets an untrusted caller distinguish "user not found" from "wrong password"; auth and authz failures return one indistinguishable response.
- **MUST** — a raw untrusted value echoed back is treated as untrusted output (length-bounded, escaped/encoded for its sink) against log-forging newlines or response-splitting. *Escape hatch: echoing a short, validated, non-sensitive field (a known enum value, a numeric id) for diagnostics is fine.*

```typescript
// WRONG — token in the log, raw SQL + stack to the client
logger.error(`auth failed with token ${token}`)
res.status(500).send(err.stack)                 // leaks internals to the caller

// RIGHT — redact internally, correlation id externally
logger.error("auth.verify failed", { correlationId, userId, cause: err })   // no token
res.status(500).json({ error: "internal_error", correlationId })            // opaque to client
```

```python
# WRONG — the message reveals which half of the credential was wrong (enumeration oracle)
if user is None:
    raise AuthError("no account for that email")
if not verify(password, user.hash):
    raise AuthError("incorrect password")

# RIGHT — one indistinguishable failure for both arms; detail only in the internal log
if user is None or not verify(password, user.hash):
    logger.warn("auth.login failed", extra={"email_hash": hash_email(email)})
    raise AuthError("invalid email or password")   # client cannot enumerate accounts
```

## Q7.10 — Aggregate Partial Failures Explicitly

- **MUST** — an operation over a collection reports a determinate outcome: full success, full failure, or an explicit partial result naming which items succeeded and which failed with their errors. **MUST NOT** — return only the successes and drop the failures.
- **MUST NOT** — concurrent fan-out lets one branch's failure silently cancel and discard the others unless cancel-on-first-failure is the documented intent; use an all-settled / `awaitAll`-with-results form when every outcome matters.
- **MUST** — a partial failure that leaves persistent state half-written is reported as partial (not success) AND either rolls back or records a compensating/repair action (Data Integrity Q13).
- **SHOULD** — validation accumulates all field errors before returning. *Escape hatch: a pipeline where a later step is meaningless after an earlier failure (parse before semantic-check) MAY short-circuit; document that ordering is intentional.*

```typescript
// WRONG — Promise.all rejects on the first failure; the other results are lost
const saved = await Promise.all(items.map((i) => save(i)))   // one reject discards the rest

// RIGHT — settle every branch; report successes and failures explicitly
const results = await Promise.allSettled(items.map((i) => save(i)))
const failures = results.flatMap((r, idx) =>
  r.status === "rejected" ? [{ item: items[idx], error: r.reason }] : [])
if (failures.length > 0) {
  logger.warn("batch save partial failure", { failed: failures.length, total: items.length })
}
return { saved: results.length - failures.length, failures }
```

## Q7.11 — Timeouts, Deadlines, and Cancellation Are Typed Errors

- **MUST** — every external/asynchronous call is bounded by an explicit timeout (Resource Management Q12.3), and the expiry surfaces as a distinct, typed `Timeout`/`DeadlineExceeded` domain error — not a hang, not a `null`, not a generic `Unavailable`.
- **MUST** — a timeout error carries whether the operation is safe to retry: a read past deadline is typically retriable; a non-idempotent write past deadline is NOT retriable without an idempotency key (Q7.12), because the write may have committed server-side after the client gave up.
- **MUST** — a deadline is propagated to downstream calls (shrinking budget), not reset at each hop. **MUST NOT** — pass a fresh full timeout downstream after most of the budget is spent. *Escape hatch: the outermost entry point with no inbound deadline, and a deliberately detached/background job given its own budget, MAY set a fresh budget; the detachment MUST be documented at the call site.*
- **MUST** — cancellation observed by the boundary surfaces as the cancellation error (Q7.8), not reclassified as a fault that gets logged at `error` or retried.

```kotlin
// WRONG — timeout is indistinguishable from any other failure; caller cannot decide to retry
suspend fun load(id: String): User =
    withTimeout(2_000) { client.fetch(id) }   // TimeoutCancellationException leaks raw

// RIGHT — map the deadline to a typed, retriable domain error carrying the budget.
// NOTE: withTimeout throws TimeoutCancellationException, a SUBCLASS of CancellationException,
// so it cannot go through safeRun (whose first arm re-throws CancellationException, Q7.1).
// Catch the timeout subtype FIRST, then re-throw any genuine cancellation (Q7.8) — this is
// the sanctioned extra try/catch site of Q7.1 (foreign API safeRun cannot wrap cleanly).
suspend fun load(id: String): SafeResult<User> =
    try {
        SafeResult.Success(withTimeout(remainingBudget()) { client.fetch(id) })
    } catch (e: TimeoutCancellationException) {                                      // subtype first
        SafeResult.Failure(UserUnavailable(retriable = true, cause = e))            // read: retriable
    } catch (e: CancellationException) {
        throw e                                                                      // never reclassify (Q7.8)
    } catch (e: Throwable) {
        SafeResult.Failure(UserUnexpected(e))
    }
```

```go
// WRONG — caller cannot tell a deadline from a real failure; errors.Is check is missing
if err != nil {
    return Unavailable{}
}

// RIGHT — deadline is a distinct, inspectable error; non-idempotent writes are NOT marked retriable
if errors.Is(err, context.DeadlineExceeded) {
    return TimeoutError{Op: "charge", Retriable: false}   // write may have committed server-side
}
```

## Q7.12 — Retry Only Idempotent Operations

- **MUST** — a retry wrapper (Resource Management Q12.5) only wraps idempotent operations, OR operations carrying a client-generated idempotency key the server deduplicates. **MUST NOT** — retry a non-idempotent, unkeyed mutation.
- **MUST** — a `Timeout`/`Unavailable` on a non-idempotent write is surfaced as ambiguous ("may or may not have applied"); the caller resolves ambiguity by reconciliation or an idempotency key, never by hopeful re-submission.
- **MUST** — the retriable predicate excludes permanent errors (validation, not-found, auth, conflict) and cancellation/fatal signals (Q7.8).
- **SHOULD** — an idempotency key is derived from stable inputs (request id, content hash) so a retry presents the same key. *Escape hatch: a naturally idempotent operation (a PUT that fully replaces, a set-to-value, a delete-by-id) needs no key; document the idempotency so a future reader does not add a redundant one.*

```typescript
// WRONG — timeout retries the charge; the first attempt may have already captured funds
await withRetry(() => api.charge(amount, card), { maxAttempts: 3, isRetriable: () => true })

// RIGHT — same idempotency key on every attempt; the server dedupes; predicate excludes 4xx
const key = `charge:${orderId}`               // stable across retries
await withRetry(
  () => api.charge(amount, card, { idempotencyKey: key }),
  { maxAttempts: 3, baseMs: 100, isRetriable: (e) => e instanceof Unavailable && e.retriable },
)
```

```python
# RIGHT — a non-idempotent unkeyed write surfaces ambiguity instead of being retried.
# safe_run returns the (value, err) 2-tuple defined in Q7.1 — unpack it, do not access `.error`.
value, err = safe_run(lambda: payments.capture(order_id))
if isinstance(err, TimeoutError):
    # do NOT retry blindly — reconcile, because the capture may have committed server-side
    raise AmbiguousOutcome("capture may have applied; reconcile via payments.get") from err
```

## Q7.13 — Circuit-Break and Shed, Don't Retry-Storm

- **MUST** — a boundary that retries or fans out to a shared dependency is protected by a circuit breaker (or equivalent load-shed / bulkhead) so that, once failures exceed a threshold, calls fail fast locally instead of queuing against the dead dependency. (Reinforces Resource Management Q12.5 / Q12.8.)
- **MUST** — when the breaker is open, fall back to the rule already governing the path: fail closed for a gating check (Q7.7), a documented safe default for a recoverable read (Q7), or a typed `Unavailable` otherwise. **MUST NOT** — silently return stale-as-fresh or empty-as-complete.
- **MUST** — honor a `429`/`RateLimited`/`Retry-After` response, backing off for at least the indicated interval. **MUST NOT** — retry sooner or hammer through a throttle as a generic retriable error.
- **SHOULD** — breaker thresholds, open duration, and half-open probe count are judgment calls; any values are acceptable if they carry a one-line rationale. *Escape hatch: a boundary that calls a strictly local, in-process, non-shared resource (a pure CPU computation, an in-memory map) needs no breaker.*

```typescript
// WRONG — every caller keeps retrying the dead service; the retries are the outage
async function rate(): Promise<Quote> {
  return withRetry(() => quotes.fetch(), { maxAttempts: 5, isRetriable: () => true })
}

// RIGHT — breaker fails fast when the dependency is unhealthy; typed fallback, honor Retry-After
async function rate(): Promise<SafeResult<Quote>> {
  if (breaker.isOpen()) return { success: false, error: new QuotesUnavailable({ retriable: true }) }
  const r = await safeRunAsync(() => quotes.fetch())
  if (!r.success) {
    breaker.recordFailure()
    // Surface the backoff interval to the retry layer that owns the decision to wait/retry;
    // do NOT sleep here — this path fails fast and does not retry, so a sleep would only stall
    // the caller's failure response with no benefit. Honoring Retry-After = the retry wrapper
    // waits at least retryAfterMs before re-attempting.
    const retryAfterMs = r.error instanceof RateLimited ? r.error.retryAfterMs : undefined
    return { success: false, error: new QuotesUnavailable({ retriable: true, retryAfterMs, cause: r.error }) }
  }
  breaker.recordSuccess()
  return r
}
```

## Q7.14 — Report Outcomes to the User Truthfully

- **MUST NOT** — a user-facing action that performs a mutation reports success until the mutation is confirmed durable, OR **MUST** make the optimistic nature explicit and reconcile/roll back visibly on failure. **MUST NOT** — a silent revert that leaves the optimistic UI showing the never-applied change.
- **MUST** — an error shown to a user is actionable and honest (what failed, and whether to retry) without leaking internals (Q7.9). **SHOULD NOT** — a generic "Something went wrong" with no recovery affordance and no correlation id be the only signal; show a correlation id the user can quote. *Escape hatch: a transient, auto-recovering failure (a retry succeeded behind the scenes) need not be surfaced at all.*
- **MUST** — a background failure the user can do nothing about (a best-effort sync) stays observable to operators (Q7.5) even when correctly hidden from the user; hidden-from-user is never hidden-from-monitoring.

```typescript
// WRONG — shows success immediately, then silently reverts on failure; user thinks it saved
function onSave(draft: Draft) {
  setStatus("Saved")                 // optimistic
  api.save(draft).catch(() => setDraft(previous))   // silent revert — user never learns it failed
}

// RIGHT — optimistic but honest: failure is surfaced, retryable, and observable to operators
async function onSave(draft: Draft) {
  setStatus("Saving…")
  const r = await safeRunAsync(() => api.save(draft))
  if (r.success) { setStatus("Saved"); return }
  logger.error("draft.save failed", { draftId: draft.id, cause: r.error })   // operator-visible
  setStatus("Couldn't save — tap to retry")                                   // user-honest + actionable
  setDraft(draft)                                                             // keep their edits, do not discard
}
```

## Self-Review Checklist

| # | Check | Required | Rule |
|---|---|---|---|
| 1 | Every trust boundary converts failures to error-as-value; no unhandled rejection/future/error channel/callback | Required | Q7 |
| 2 | "Recoverable" is a documented decision (not convenience); recoverable returns a safe default, else propagates typed | Required | Q7 |
| 3 | No catch mid-call-chain; catch at the boundary or let propagate; no log-and-rethrow | Required | Q7 / Q7.4 |
| 4 | Generic `Exception`/`Error`/`Throwable`/`catch(...)`/bare `except:` caught only inside the `safeRun` body | Required | Q7 / Q7.1 |
| 5 | Catches name the narrowest producible error type (broad catch only inside `safeRun` / open-ended foreign callback) | Required | Q7 |
| 6 | Business logic contains no raw try/catch; boundaries call `safeRun` and branch on `SafeResult` | Required | Q7.1 |
| 7 | No `SafeResult` discarded; both success and failure arms exhaustively acted upon | Required | Q7.1 |
| 8 | Async boundaries use `safeRunAsync`; no synchronous `safeRun` over an un-awaited Promise/future/lazy | Required | Q7.1 |
| 9 | No ad-hoc `runCatching`/`runSafeCatching`/`try?`/`try!` in business logic; no nested `safeRun` re-wrap | Required | Q7.1 |
| 10 | Any extra raw try/catch site names its (measured, if perf) constraint inline and still maps to a domain type | Required | Q7.1 |
| 11 | Go errors checked (no unjustified `_` discard); recovered panic returned as error | Required | Q7.1 |
| 12 | Rust: no `unwrap`/`expect`/`panic!`/`unreachable!`/unchecked index on production boundary; no silent `let _ =` drop | Required | Q7.1 |
| 13 | Low-level errors mapped to a closed set of domain error types at the boundary | Required | Q7 / Q7.2 |
| 14 | Mapping is exhaustive; unrecognized low-level errors hit an explicit non-retriable `unknown`/`unexpected` variant | Required | Q7.2 |
| 15 | Retriability (and other machine signals) encoded in the error type/code, never parsed from a free-text message | Required | Q7.2 |
| 16 | Wrapping preserves the cause chain (`cause`/`from e`/`%w`/`#[source]`) and the stack trace | Required | Q7.3 |
| 17 | Cause not collapsed to a string where structured unwrapping (`errors.Is`/`__cause__`/`instanceof`) is used | Required | Q7.3 |
| 18 | Failure logged once at the handling boundary, structured (op, correlation id, code, cause) | Required | Q7.4 |
| 19 | Log context is structured key/value (SHOULD), not interpolated into a free-text message | Required | Q7.4 |
| 20 | Severity matches consequence; genuine fault not downgraded to dodge an alert; not logged at debug/info | Required | Q7.4 / Q7.5 |
| 21 | No empty `catch`/`.catch(()=>{})`/`except: pass`/`.catch(noop)`; every catch recovers, propagates, or logs error/warn | Required | Q7.5 |
| 22 | Deliberate swallow carries an inline justification and is not on an access/money/mutation path | Required | Q7.5 |
| 23 | `finally` does not mask the in-flight exception; cleanup error is suppressed/secondary, primary propagates | Required | Q7.5 |
| 24 | Best-effort failures still log at (possibly sampled) warn and are marked best-effort at the call site | Required | Q7.5 |
| 25 | Expected outcomes modeled as values (Optional/Result/sum type), not thrown-and-caught control flow | Required | Q7.6 |
| 26 | No overloaded sentinel (`-1`/`""`/empty/`NaN`/`null`) conflating absence/failure with a valid value | Required | Q7.6 |
| 27 | Gating checks fail closed (deny/reject/safe restrictive default) on error/timeout/ambiguity; never fail open | Required | Q7.7 |
| 28 | Partial result from a security/correctness check is not treated as a full pass | Required | Q7.7 |
| 29 | Fail-closed error path is covered by a fault-injection test asserting denial | Required | Q7.7 |
| 30 | Broad handlers re-throw cancellation (`CancellationException`/`CancelledError`/`InterruptedException`+flag/`ctx.Err()`) | Required | Q7.8 |
| 31 | Fatal errors (`OOM`/`StackOverflow`/`LinkageError`/`SystemExit`/`KeyboardInterrupt`) not caught by boundary handlers | Required | Q7.8 |
| 32 | Retry predicate excludes cancellation/fatal signals | Required | Q7.8 / Q7.12 |
| 33 | No secrets/PII/raw request bodies/whole error or request objects in messages, logs, or payloads | Required | Q7.9 |
| 34 | Remote/public errors map to a client-safe message + opaque correlation id; no stack/SQL/path/version leak | Required | Q7.9 |
| 35 | Auth/authz failures are indistinguishable (no enumeration oracle via message/status/timing) | Required | Q7.9 |
| 36 | Echoed untrusted values are bounded and encoded for their sink | Required | Q7.9 |
| 37 | Collection operations report a determinate outcome (full success/failure or explicit partial) | Required | Q7.10 |
| 38 | Concurrent fan-out settles all branches when every outcome matters; no silent loss of siblings | Required | Q7.10 |
| 39 | Partial persistent write reported as partial and rolled back or compensated, never reported as success | Required | Q7.10 |
| 40 | Every external/async call has an explicit timeout surfaced as a typed `Timeout`/`DeadlineExceeded` error | Required | Q7 / Q7.11 |
| 41 | Timeout error carries retriability; non-idempotent write past deadline is ambiguous, not cleanly retriable | Required | Q7.11 / Q7.12 |
| 42 | Deadline budget is propagated (shrinking) downstream, not reset full at each hop (entry point / detached job MAY set a fresh budget, documented) | Required | Q7.11 |
| 43 | Retries wrap only idempotent ops or idempotency-keyed ops; non-idempotent unkeyed mutation not retried | Required | Q7 / Q7.12 |
| 44 | Retry predicate excludes permanent errors (validation/not-found/auth/conflict) | Required | Q7.12 |
| 45 | Boundaries to shared dependencies are circuit-broken / bulkheaded; open-breaker fallback obeys Q7.7/typed-`Unavailable` | Required | Q7 / Q7.13 |
| 46 | `429`/`Retry-After`/throttle honored; not hammered as a generic retriable error | Required | Q7.13 |
| 47 | User-facing mutations report success only when confirmed (or optimistic + visible reconcile/rollback); no silent revert | Required | Q7.14 |
| 48 | User errors are honest and actionable (retry hint + correlation id); background failures stay operator-visible | Required | Q7.14 |
| 49 | Retry logic in a dedicated wrapper with a finite ceiling and retriable predicate (MUST); backoff/jitter strategy per Q12.5 (SHOULD + escape hatch); no inline retries | Required | Q7 / Q12.5 |
