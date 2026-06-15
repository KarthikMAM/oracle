# Documentation

Doc comments are the contract between author and caller; a missing, vague, or stale one forces every reader to reverse-engineer intent from implementation — and a doc that *lies* is worse than none, because callers trust it and ship the bug it described away.

## Anchors

- **Knuth — Literate Programming (1984)** — code is written for humans first, machines second; the explanation and the code are one artifact
- **Hyrum's Law (Wright)** — every observable behavior of an interface eventually becomes a contract; doc comments are how you fix what is *contract* versus what is *accident*
- **Joel Spolsky — Things You Should Never Do (2000)** — code is read an order of magnitude more often than it is written; optimize for the reader
- **Parnas — On the Criteria To Be Used in Decomposing Systems into Modules (1972)** — a module's documented interface is the only thing callers may depend on; everything else is free to change
- **Bloch — How to Design a Good API and Why It Matters (2006)** — document every public element: every class, method, parameter, and exception; document state space and the threading contract
- **Lamport — Specifying Systems (2002)** — a precise statement of the boundary behavior (what is required of the caller, what is guaranteed in return) is what makes a component composable; prose hand-waving is not a specification

## Q2.1 — Doc Comments

- **MUST** — every function (public or private) carries a doc comment (private trivial symbols may omit it, subject to the escape hatch at the end of this section) with: purpose in one sentence (what, not how); every parameter (meaning, range/domain, units, nullability/optionality, ownership/lifetime, cross-parameter constraints); return value (meaning, guarantees, units, ownership, edge-case values); every thrown error and the distinct condition producing each; side effects (logging, metrics, state mutation, I/O, retries, event emission), and a pure function whose name suggests otherwise states it is side-effect-free; concurrency contract when reachable from more than one thread; why the function exists when the name does not make it obvious.
- **MUST** — item 2 (parameters) applies to every parameter without exception: generic type params with non-trivial bounds, varargs, default-valued params (state what the default means), and the associated values of every error case.
- **MUST** — a parameter or return carrying units, currency, time zone, encoding, radix/base, coordinate system, or scale/precision states it, because the type alone (`Int`, `Double`, `String`, `Long`) does not.
- **MUST NOT** — restate the signature in prose (`/** Sets the name. @param name the name */`); replace with the actual contract or a self-evident signature.
- **SHOULD** — state what the function deliberately does *not* do when a reasonable caller would assume it does. *Escape hatch: omit non-goals when the purpose sentence and parameter contract already foreclose the assumption.*
- **MUST** — every type (class, struct, enum, protocol, interface, non-obvious type alias) carries a doc comment stating responsibility, invariants, mutability/thread-safety posture, and any no-instantiate/no-subclass/no-implement restriction (e.g. "obtain via `Factory.create`; do not construct directly").
- **MUST** — use the language's documentation comment form (`///` Swift/Rust, `/** */` Kotlin/Java/TypeScript, `"""` Python docstring, doc-comment block Go), not an ordinary comment, so tooling indexes it; the comment sits directly on the declaration with no intervening blank line.
- **SHOULD** (items 1–6 are MUST) — *Escape hatch: a trivial, self-evident, private symbol whose name and signature fully express the contract and which has none of (constrained params, units, nullability ambiguity, error cases, side effects, concurrency exposure) MAY omit the doc comment — e.g. a private `isEmpty(): Boolean` over an obvious field. The moment any one of those appears, the full doc comment is mandatory again.* Public API has no escape hatch.

### Examples

```swift
// WRONG — units, range, nullability, failure modes, and side effects all left to the caller to guess
/// Fetches a user.
func fetchUser(id: String) async throws -> User?

// RIGHT — purpose, param contract, return meaning, errors, side effect, freshness non-goal, and threading are explicit
/// Fetches a user by ID, hitting the local cache first.
///
/// - Parameter id: User identifier. Non-empty. Treated as opaque (never parsed).
/// - Returns: The user if found in cache or storage. `nil` if the user does not exist.
/// - Throws: `NetworkError.timeout` if the storage backend exceeds the configured deadline.
///           `AuthError.unauthenticated` if the caller's session has expired.
/// - Note: Side effect: emits a `cache.hit` or `cache.miss` analytics event per call.
/// - Note: Does NOT refresh an expired cache entry; call `refresh(id:)` first if freshness matters.
/// - Note: Concurrency: `nonisolated` — callable from any isolation context. Per Swift's
///         model, execution may resume on a different executor after the internal `await`;
///         the caller's actor context is restored on return, not the original thread.
func fetchUser(id: String) async throws -> User?
```

```typescript
// WRONG — a plain comment is invisible to doc tooling and to the caller's editor tooltip
// Returns the active session, or null if none.
export function currentSession(): Session | null

// RIGHT — a real doc comment surfaces in tooltips, generated docs, and reference checkers
/** Returns the active session, or `null` if no user is signed in. Never throws. */
export function currentSession(): Session | null
```

## Q2.2 — Inline Comments

- **MUST** — explain *why*, not *what*; the exception is a genuinely non-obvious *what* (a bit-twiddling trick, a regex, a magic constant), which MAY be explained, paired with the *why*.
- **MUST** — remove commented-out code (any-line rule, even a single statement). *Exception: a commented line that is itself an example explicitly referenced by an adjacent comment (`// example invocation: foo(2, "ms")`).*
- **MUST NOT** — commit commented-out tests or assertions; use the framework's skip/ignore with a linked ticket so the suite reports it as pending.
- **MUST** — `// TODO` / `// HACK` / `// FIXME` / `// WORKAROUND` / `// XXX` / `// BUG` / `// REFACTOR` / `// TEMP` (and any equivalent backlog marker) include a linked tracking ticket ID; **SHOULD** name an owner. A marker without a ticket is a violation.
- **MUST** — remove obvious comments restating the code (`i += 1 // increment i`).
- **MUST** — put comments on the line ABOVE the code they describe. *End-of-line comments MAY be used only for a short unit/enum annotation on a data declaration (`val timeoutMs = 30_000 // milliseconds`).*
- **MUST** — correct or delete a comment that contradicts the code it sits on, in the same change (composes with Q2.3).
- **MUST** — name the specific rule and justify it for every `// eslint-disable*`, `@ts-ignore` / `@ts-expect-error`, `// noqa`, `@Suppress`, `#[allow(...)]`, `// nolint`, or `// swiftlint:disable`; **MUST NOT** use a blanket file-wide suppression where a single-line, single-rule one would do.

### Examples

```kotlin
// WRONG — restates what the code does
val timeout = 30.seconds // sets timeout to 30 seconds

// RIGHT — explains why
// Covers typical network latency plus retry backoff
val timeout = 30.seconds

// WRONG — orphan TODO with no ticket and no owner
// TODO: handle pagination

// RIGHT — TODO with ticket link and owner
// TODO(asmith): handle pagination — see https://issues.example.com/PROJ-1234
```

```typescript
// WRONG — blanket suppression hides this AND every future violation in the file
/* eslint-disable */
const raw: any = JSON.parse(body)

// WRONG — a silenced assertion masquerading as a comment; the alarm is gone
// expect(balance).toBeGreaterThanOrEqual(0)

// RIGHT — single-rule, single-line suppression with the reason it is sound here
// eslint-disable-next-line @typescript-eslint/no-explicit-any -- upstream SDK types `data` as any; validated by zod on the next line
const raw: any = sdk.event.data
const parsed = EventSchema.parse(raw)
```

## Q2.3 — Doc/Code Agreement and Maintenance

- **MUST** — a change altering a function's signature, return values, thrown errors, side effects, units, nullability, or concurrency contract updates that function's doc comment in the *same* change; **MUST NOT** ship a behavior change with a now-false doc comment.
- **MUST NOT** — describe behavior that does not exist (an unimplemented parameter, a "future" return, a `@throws` for an exception no longer raised); document what the code does today.
- **MUST** — a reference in a comment to a symbol, file, ticket, or URL resolves; update or remove one pointing at a deleted function or dead link when discovered.
- **MUST** — rewrite a copied doc comment for the function it now sits on; one still naming the *original* function's params, return, or behavior is a lie.
- **SHOULD** — reference symbols using the language's linkable syntax (`[Type]` / `{@link Type}` / `` `Type` ``) so renames and reference checkers catch drift mechanically. *Escape hatch: prose references to symbols outside the current module's reachable scope (a remote service's field, a protocol wire name) MAY remain plain text.*

### Examples

```typescript
// WRONG — signature changed to return null on miss, but the doc still promises a throw
/**
 * @returns The order.
 * @throws {NotFoundError} If no order exists for the id.
 */
function findOrder(id: OrderId): Order | null {
  return cache.get(id) ?? null   // it no longer throws — the doc is now a lie
}

// RIGHT — doc updated in lockstep with the behavior
/**
 * @returns The order, or `null` if no order exists for the id.
 */
function findOrder(id: OrderId): Order | null {
  return cache.get(id) ?? null
}
```

## Q2.4 — Document Failure, Boundary, and Edge-Case Behavior

- **MUST** — state the behavior for every edge case the function distinguishes, where applicable: empty/zero/absent input; boundary values (min/max accepted, and what happens at and just past each — rejected, clamped, throws); numeric overflow, precision, and rounding (wraps, saturates, throws, rounds half-up); duplicate or conflicting input (last-wins, first-wins, reject, merge); not-found / no-match (empty vs. `null` vs. throw); failure atomicity on partial failure (rolled back, left in place, or undefined); idempotency and retry safety for any state-mutating or I/O function; ordering and stability of any returned collection/sequence (composes with Q2.5).
- **MUST** — a function that *silently* clamps, truncates, rounds, swallows, coerces a type, or drops a duplicate documents that behavior at the point of surprise.
- **MUST** — a function whose result depends on ambient state (current time, locale, time zone, environment, a global, a random source, the current user, a feature flag) documents that dependency.
- **SHOULD** — a function with a non-obvious cost cliff (super-linear complexity, unbounded buffer, synchronous network call, full table scan) documents the cliff and its trigger. *Escape hatch: omit the cost note when the cost is obvious from name and signature (`sortInPlace`, `readEntireFile`); detailed asymptotics belong to the performance domain.*
- **MUST** (escape-hatch-bounded) — *Escape hatch: a function that genuinely distinguishes no edge cases (a pure total function over its whole input domain, e.g. `not(b: Bool): Bool` or `isEmpty(s: String): Bool`, where every input maps to a defined result with no boundary, overflow, or absent case) need only state that totality.* Reaching for this on anything touching collections, I/O, parsing, time, money, or concurrency is almost always a mistake.

### Examples

```python
# WRONG — silent clamping is invisible; a caller passing 500 gets 100 and never knows
def fetch_page(limit: int) -> list[Row]:
    return _query(min(limit, 100))

# RIGHT — the surprising boundary behavior is documented at the point of surprise
def fetch_page(limit: int) -> list[Row]:
    """Return up to `limit` rows from the current page.

    Args:
        limit: Maximum rows to return, in 1..=100. Values above 100 are
            CLAMPED to 100 (the server's hard page cap); values below 1
            raise ValueError. Passing 500 returns 100 rows, not 500.

    Raises:
        ValueError: If `limit` < 1.
    """
    if limit < 1:
        raise ValueError("limit must be >= 1")
    return _query(min(limit, 100))
```

```kotlin
// WRONG — overflow on a long accumulation is silent; the total wraps negative at scale
/** Returns the total bytes across all chunks. */
fun totalBytes(chunks: List<Chunk>): Long

// RIGHT — empty-input result, units, and the overflow contract are all explicit
/**
 * Returns the total size across all chunks, in BYTES.
 *
 * @param chunks Chunks to sum. May be empty (returns 0).
 * @return Sum of chunk sizes. THROWS [ArithmeticException] on overflow rather than
 *         wrapping silently, because a wrapped negative total has caused quota-bypass bugs.
 */
fun totalBytes(chunks: List<Chunk>): Long  // uses Math.addExact internally
```

## Q2.5 — Stability, Deprecation, and Hyrum's Law Boundaries

- **MUST** — a public symbol whose behavior callers MUST NOT depend on (set iteration order, exact wording of a non-localized message, the fact that a cache currently never misses, an auto-generated id's precise value) says so explicitly ("order is unspecified", "for diagnostics only — do not parse").
- **MUST** — a no-longer-recommended symbol is marked with the language's deprecation mechanism (`@Deprecated` / `@deprecated` / `@available(*, deprecated:)` / `#[deprecated]`) and names *the replacement* and *the removal plan*. For a symbol on the published/public API surface the removal plan MUST state a concrete *version* (the earliest version removal MAY occur), per api-design.md Q5.3 — a tracking ticket alone is insufficient there. A ticket-only removal plan is allowed only for non-published / internal symbols.
- **MUST** — experimental or unstable API that may change without the project's normal compatibility guarantee is marked as such (`@Experimental`, `@beta`, a documented `Unstable` annotation); conversely, once a symbol is in the stable public surface, a change to its documented contract MUST be treated as a breaking change and called out in the change that makes it.
- **MUST** — pre-conditions the caller must satisfy (hold a lock, call `open()` first, not call from the main thread, call within a transaction) are documented as pre-conditions, not buried in prose; **SHOULD** likewise state guaranteed post-conditions and invariants (the handle is closed, the list is sorted, the lock is released).

### Examples

```kotlin
// WRONG — deprecated with no replacement and no removal date; callers are stranded
@Deprecated("Don't use this anymore")
fun parseLegacy(raw: String): Config

// RIGHT — names the replacement and a concrete removal version (api-design.md Q5.3);
// tooling can auto-migrate. The tracking ticket is supplementary, not the removal plan itself.
@Deprecated(
    message = "Ambiguous on malformed input; use parseConfig which returns a typed Result. " +
        "Removal no earlier than 4.0.0; migration tracked in https://issues.example.com/PROJ-3100.",
    replaceWith = ReplaceWith("parseConfig(raw)"),
)
fun parseLegacy(raw: String): Config
```

```swift
// RIGHT — pre- and post-conditions stated, not buried in prose
/// Appends a record to the open segment.
///
/// - Precondition: `open()` has been called and `close()` has not; calling on a
///   closed segment traps. Not safe to call concurrently with `close()`.
/// - Postcondition: On return, the record is durably flushed to disk (fsync completed).
func append(_ record: Record)
```

## Q2.6 — Examples Must Be Honest and Current

- **MUST** — a code example in a doc comment, README, or guide reflects the current API: it compiles against, and behaves as described with, the version it ships beside; one using a renamed, removed, or re-signatured symbol is fixed in the same change.
- **SHOULD** — examples are runnable and verified by the build where the language supports it (Rust `///` doctests, Python `doctest`, compiled snippet tests). *Escape hatch: an intentionally partial snippet (`// ... configure the client ...`) is allowed when a full example would obscure the point; mark the elision so a reader does not copy it as complete.*
- **MUST NOT** — show an insecure or unsafe usage (hardcoded credential, disabled TLS verification, swallowed error, SQL string interpolation, `eval` of untrusted input, missing resource cleanup) as if it were the recommended pattern.
- **MUST** — an example demonstrating a fallible operation shows the error being handled (or explicitly notes that handling is elided for brevity and what the real handling would be).

### Examples

```python
# WRONG — example "works" but models an unsafe pattern readers will copy
def connect(dsn: str) -> Conn:
    """Open a database connection.

    Example:
        >>> connect("postgres://admin:hunter2@db/prod")  # hardcoded prod credential
    """

# RIGHT — models the safe, current pattern; the live-resource lines are marked so
# `doctest` skips them rather than NameError-ing on `settings` or opening a real DB
def connect(dsn: str) -> Conn:
    """Open a database connection.

    Read the DSN from configuration; never hardcode credentials.

    Example:
        >>> dsn = settings.database_dsn()   # doctest: +SKIP  (loaded from env/secrets manager)
        >>> conn = connect(dsn)             # doctest: +SKIP  (opens a live connection)
        >>> conn.is_open                    # doctest: +SKIP
        True
    """
```

## Q2.7 — No Secrets, Personal Data, or Internal Leaks in Comments or Docs

These are safety MUSTs and are **not** waivable by the exception block in Q2.10.

- **MUST NOT** — a comment, doc comment, example, or committed doc file contain a live secret: API key, password, token, private key, connection string with credentials, session cookie, or signing material. Examples use obvious placeholders (`<API_KEY>`, `xxxx`, `example.com`).
- **MUST NOT** — embed real personal data (a real customer's email, phone, address, government ID, payment instrument, or full name tied to an account). Use synthetic data (`jane.doe@example.com`, `555-0100`, `4111 1111 1111 1111`).
- **MUST NOT** — paste a real captured request/response body, log line, or stack trace that may carry embedded tokens or PII; reproduce only a redacted, synthetic equivalent.
- **SHOULD NOT** — hardcode internal-only detail that aids an attacker (a production hostname, internal IP, bucket name, queue ARN, exact infrastructure topology, or an unpatched-vulnerability description) into comments or examples; refer to it abstractly or via configuration.

### Examples

```typescript
// WRONG — a live token and a real customer email committed forever in history
// works with prod: Authorization: Bearer sk_live_EXAMPLE_leaked_in_a_comment
// repro with customer jane.doe@example.com, acct 4471203

// RIGHT — placeholder secret and synthetic data
// Send `Authorization: Bearer <API_TOKEN>` (token from the secrets manager, never inlined).
// Repro with the synthetic fixture user jane.doe@example.com.
```

## Q2.8 — Module, Package, and Entry-Point Documentation

- **MUST** — every public module/package/library carries a module-level doc comment or README stating, at minimum: its responsibility (what problem it solves), the primary entry points (the few symbols a new caller starts from), and any cross-cutting contract for the whole module (threading model, error-as-value convention, required initialization order). Use the language's module-doc mechanism (Rust `//!`, Python module `"""docstring"""`, Java/Kotlin `package-info` / module doc, a top-of-file doc block, or a package `README`).
- **MUST** — a module requiring a specific initialization or lifecycle order (configure before first use, call `start()` before any operation, dispose explicitly) documents that order at the module level.
- **MUST** — the module doc distinguishes the public surface from internal helpers when the language does not enforce it; where visibility modifiers exist (`internal`, `private`, `pub(crate)`, package-private) they are used to make "not part of the public API" mechanical rather than documentary (Parnas).
- **SHOULD** — the overview links to a runnable example or quick-start (composes with Q2.6). *Escape hatch: a single-purpose internal module whose name and sole public symbol are self-evident (`StringCaseUtil`) MAY omit the overview; the moment it grows a second entry point or a lifecycle requirement, the overview is mandatory.*

### Examples

```rust
// WRONG — public crate root with no orientation; callers guess the entry point
pub mod client;
pub mod retry;
pub mod transport;

// RIGHT — module-level doc states purpose, entry point, and the cross-cutting contract
//! Async HTTP client for the Billing API.
//!
//! Start with [`client::BillingClient::connect`]; all other types are reached from it.
//! Every operation returns `Result<_, BillingError>` — this crate never panics on a
//! recoverable failure. Construct one client per process and share it (it is `Send + Sync`
//! and pools connections internally); constructing one per request defeats pooling.
pub mod client;
pub mod retry;
pub mod transport;
```

```python
# RIGHT — module docstring states the lifecycle contract up front
"""Tenant-scoped data access for the reporting service.

Entry point: `Session.open(tenant_id)`. Everything else hangs off a Session.

Lifecycle (MUST be observed in this order):
    1. `configure(settings)` once at process start, before any Session is opened.
    2. `Session.open(tenant_id)` per request; use as a context manager so it closes.
Calling `Session.open` before `configure` raises `NotConfiguredError`.

All Session methods are thread-compatible but NOT thread-safe: do not share one
Session across threads; open one per unit of work.
"""
```

## Q2.9 — Inclusive, Precise, and Tooling-Visible Language

- **MUST** — comments, doc comments, and committed docs use inclusive terminology: *primary/replica*, *leader/follower*, *allowlist/denylist*, *blocklist* in place of the exclusionary equivalents. *Quoting a third-party API's literal identifier that uses an excluded term is the one allowance, and SHOULD be wrapped in backticks to mark it as a verbatim external name.*
- **MUST NOT** — depend on ephemeral context the reader cannot recover ("the new way", "as discussed", "see the other PR", "temporary fix for now", "recent change"); state the durable fact or link a permanent reference.
- **MUST** — doc-lint / doc-coverage tooling, where the project runs it (TypeScript `tsdoc`, Kotlin/Java doc warnings as errors, Rust `#![warn(missing_docs)]` on public crates, Python docstring linters), passes; **MUST NOT** exempt a public symbol from doc-coverage by a blanket suppression (targeted, justified suppression is governed by Q2.2 and Q2.8).
- **SHOULD** — prose is precise where precision is free ("returns at most N items" over "returns some items"; "MUST be non-negative" over "should be positive-ish"; a concrete unit over "a short delay"). *Escape hatch: genuinely open design intent ("callers SHOULD prefer the batched form for large inputs") MAY stay qualitative; reserve vagueness for advice, never for the contract.*

### Examples

```kotlin
// WRONG — exclusionary term, and a reference that expires the moment the PR merges
/** Adds the host to the master whitelist. See the recent change for why. */
fun allow(host: String)

// RIGHT — inclusive vocabulary; durable rationale instead of ephemeral context
/**
 * Adds [host] to the connection allowlist.
 *
 * Hosts not on the allowlist are rejected at the TLS layer (fail-closed);
 * rationale tracked in https://issues.example.com/PROJ-5012.
 */
fun allow(host: String)
```

```typescript
// WRONG — imprecise where precision is free; two callers will read it two ways
/** Waits a bit, then returns some of the results. */
function poll(): Promise<Result[]>

// RIGHT — exact unit and exact bound; one unambiguous mental model
/**
 * Polls once after a fixed 200 ms delay.
 *
 * @returns At most `BATCH_MAX` (50) results, oldest first. Empty array if none are ready
 *   (never `null`). Does not block waiting for a full batch.
 */
function poll(): Promise<Result[]>
```

## Q2.10 — Documented Exception Block

- **MUST** — a symbol or file that knowingly deviates from a SHOULD/MUST in this domain carries an exception block adjacent to the deviation containing all of: (1) **which rule** (the Q2.x id); (2) **why** the conforming path is not possible right now; (3) **scope** — exactly which symbol(s)/file(s) it covers; (4) **mitigation** — compensating documentation or control (a link to the generator, a pointer to upstream docs, a covering test); (5) **expiry** — a linked tracking ticket and a date (omit only for permanently-exempt categories such as generated code, stated explicitly).
- **MUST** — treat an exception block missing any required field as a violation. **MUST NOT** — waive a Q2.7 safety/correctness MUST via this block; an exception block citing Q2.7 is itself a violation.

### Examples

```kotlin
// RIGHT — generated file, permanently exempt, stated explicitly
//
// DOC EXCEPTION — Q2.1 (per-symbol doc comments)
// Why:        File is generated by the protobuf compiler; hand-written doc
//             comments would be overwritten on every regeneration.
// Scope:      All symbols in this file (build/generated/UserProto.kt) ONLY.
// Mitigation: Field semantics documented in the .proto source; see user.proto.
// Expiry:     Permanent (generated artifact) — not tracked.
```

```typescript
// RIGHT — time-boxed deviation behind a flag
//
// DOC EXCEPTION — Q2.4 (edge-case documentation)
// Why:        Pagination edge cases are still being designed; behavior will
//             change before launch, so documenting it now would be aspirational (Q2.3).
// Scope:      experimentalFetchPage() ONLY, gated behind the `paging_v2` flag.
// Mitigation: Function is unreachable in production (flag off); covered by a
//             skipped spec that documents intended behavior.
// Expiry:     https://issues.example.com/PROJ-4090 — due 2026-08-15.
```

## Self-Review Checklist

| # | Check | Required | Rule |
|---|---|---|---|
| 1 | Every new/modified function has a complete doc comment (purpose, params, return, throws, side effects, concurrency, why) | Required | Q2.1 |
| 2 | Every new/modified type documents its responsibility, invariants, mutability/thread-safety, and any no-instantiate/no-subclass restriction | Required | Q2.1 |
| 3 | Every parameter documented (including generic bounds, varargs, defaults, and error-case associated values) | Required | Q2.1 |
| 4 | Nullability/optionality stated for every parameter and return whose type does not already make it impossible | Required | Q2.1 |
| 5 | Units, currency, time zone, encoding, radix, coordinate system, and scale/precision stated wherever the type does not convey them | Required | Q2.1 |
| 6 | No doc comment that merely restates the signature; doc tooling form used (not a plain comment) | Required | Q2.1 |
| 7 | Deliberate non-goals stated where a reasonable caller would assume the function does more | Required | Q2.1 |
| 8 | Doc comments sit directly on the declaration with no intervening blank line | Required | Q2.1 |
| 9 | Public symbols documented with no escape hatch; private trivial-symbol omissions are genuinely trivial | Required | Q2.1 |
| 10 | Inline comments explain WHY (or a non-obvious WHAT), never restate the code | Required | Q2.2 |
| 11 | No commented-out code (any line); no commented-out tests/assertions | Required | Q2.2 |
| 12 | Every `TODO`/`HACK`/`FIXME`/`WORKAROUND`/`XXX`/`BUG`/`REFACTOR`/`TEMP` has a linked ticket | Required | Q2.2 |
| 13 | Comments on the line ABOVE code, except short unit/enum annotations on data declarations | Required | Q2.2 |
| 14 | Every linter/type/compiler suppression names the specific rule and justifies it; no blanket file-wide disable where a single-line one would do | Required | Q2.2 |
| 15 | Signature/behavior/nullability/units changes update the doc comment in the same change | Required | Q2.3 |
| 16 | No aspirational doc; referenced symbols/links resolve; copied docs rewritten for their new home | Required | Q2.3 |
| 17 | Edge cases documented: empty/zero, boundary, overflow/precision/rounding, duplicate/conflict, not-found, partial-failure atomicity, retry safety, ordering/stability | Required | Q2.4 |
| 18 | Silent clamping/truncation/rounding/coercion, ambient-state dependence, and surprising cost cliffs documented at the point of surprise | Required | Q2.4 |
| 19 | Non-contract behavior (iteration order, message wording, generated-id values) explicitly marked unspecified; stable-contract changes called out as breaking | Required | Q2.5 |
| 20 | Deprecated symbols name the replacement and a removal plan; experimental API is marked | Required | Q2.5 |
| 21 | Caller pre-conditions and guaranteed post-conditions/invariants documented as such (locks, ordering, threading, transactions) | Required | Q2.5 |
| 22 | Examples reflect the current API and are runnable/checked where the language supports it; elisions marked | Required | Q2.6 |
| 23 | No example shows an insecure/unsafe pattern as recommended; fallible-operation examples show (or note) error handling | Required | Q2.6 |
| 24 | No live secret, real personal data, captured payload/log/trace, or sensitive internal detail in any comment, example, or doc | Required | Q2.7 |
| 25 | Every public module/package has an orienting overview (responsibility, entry points, cross-cutting contract) | Required | Q2.8 |
| 26 | Required initialization/lifecycle order documented at the module level; public-vs-internal surface made mechanical via visibility modifiers | Required | Q2.8 |
| 27 | Inclusive terminology used (external verbatim names backticked); no ephemeral "new way"/"as discussed"/"temporary" references | Required | Q2.9 |
| 28 | Doc-lint/doc-coverage passes for public symbols (no blanket doc-coverage exemption); prose precise where precision is free | Required | Q2.9 |
| 29 | Any knowing deviation carries a 5-field exception block (rule, why, scope, mitigation, expiry); Q2.7 never waived | Required | Q2.10 |
