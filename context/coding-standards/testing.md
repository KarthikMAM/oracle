# Testing

Tests are the executable specification: without them, every refactor is a gamble and every "obvious" fix ships a regression that no one notices until a customer does.

## Anchors

- **FIRST** (Beck / Martin) — Fast, Independent, Repeatable, Self-validating, Timely
- **AAA** (xUnit Test Patterns, Meszaros, 2007) — Arrange, Act, Assert
- **Given-When-Then** (Dan North, BDD, 2006) — behavioral framing of a test
- **QuickCheck** (Claessen & Hughes, 2000) — property-based testing over generated inputs
- **Mutation Testing** (DeMillo, Lipton & Sayward, 1978) — a test suite is only as strong as the mutants it kills
- **Test Doubles** (Meszaros / Fowler, 2007) — dummy, stub, spy, mock, fake are distinct tools, not synonyms
- **Boundary-Value Analysis** (Myers — *The Art of Software Testing*, 1979) — defects cluster at the edges of input domains
- **Test Pyramid** (Cohn, 2009) — many fast unit tests, fewer integration tests, fewest slow end-to-end tests; an inverted pyramid is a slow, flaky suite

## Q14 — Test Quality Bar

Every rule below is enforced on tests for modified production code. Q14.1–Q14.6 define what a single test MUST look like (Naming, Isolation & Determinism, Body Structure, Assertion Quality, Edge-Case Coverage, Async & Concurrency Correctness). Property-based testing, mutation, coverage, regression, doubles, snapshots, fixture safety, and maintainability follow as their own Q-sections.

## Q14.1 — Naming

Canonical shape: `test_<methodUnderTest>_<scenario>_<expectedOutcome>` (e.g. `test_fetchUser_withExpiredToken_throwsUnauthorized`).

- **MUST** name both the scenario AND the expected outcome — never method-only (`test_fetchUser`), a vague mood (`test_happyPath`, `test_works`, `test_success`, `test_misc`), or a bare ticket number (`test_BUG_4821`).
- **SHOULD** keep the name true to what the body asserts; rename when you change what a test asserts.
- **SHOULD NOT** number or alphabetically sequence names to imply order (`test_step1`, `test_first`/`test_second`) — the binding rule is the independence in Q14.2.
- **SHOULD** use a framework's prose idiom (`describe`/`it`, `@DisplayName`, Spec-style `func test...`) instead of the snake-case template where that is the convention — the requirement is scenario + outcome, not the literal underscore format. *Any convention is acceptable if a reviewer can predict the failing assertion from the name alone.*

```typescript
// WRONG — names that say nothing on failure
it("works", () => { /* ... */ })
it("test user", () => { /* ... */ })

// RIGHT — describe/it idiom carrying scenario + outcome
describe("fetchUser", () => {
  it("throws Unauthorized when the token is expired", () => { /* ... */ })
  it("returns the cached user when the cache is warm", () => { /* ... */ })
})
```

## Q14.2 — Isolation & Determinism

- **MUST** be independent: no dependence on execution order, no reading state another test wrote; MUST pass in isolation, in reverse, and in parallel. **SHOULD** run the suite with order randomization (randomized seed) enabled. *A deterministic order MAY be pinned for a genuinely sequential integration scenario, but the dependency MUST then be made explicit (a single test method, or an ordered nested class with a documented reason), never left implicit across sibling methods.*
- **MUST NOT** share mutable fixtures (static/module-level mutable singletons, a reused database row, a mutated module global, a process-wide cache or registry); each test constructs its own inputs in the body or via a named factory/builder returning a fresh instance per call.
- **MUST NOT** make real network, file I/O, database, clock, randomness, env-var, working-directory, or locale/timezone access in unit tests — substitute a test double (see Q15): inject a clock, a seeded RNG, an explicit config, a virtual filesystem.
- **MUST** pin locale/timezone/charset explicitly where an assertion depends on them (formatted date, sorted-string order, number format, casing).
- **MUST** control time and randomness rather than sleep through them — no `sleep`/`Thread.sleep`/`setTimeout`-and-hope to synchronize async work; use a fake clock, a deterministic scheduler, an awaited future, or a bounded await-until-condition (see Q14.6).
- **MUST** root-cause and fix or quarantine a flaky test within the change that discovers it; **MUST NOT** paper over it with a retry annotation, a longer sleep, a wider timeout, a `// flaky, rerun` comment, or a suite-wide retry/rerun mechanism.
- **MUST** clean up any external/global state a test creates or mutates (temp files, containers, inserted rows, registered hooks, monkeypatched globals, signal handlers, feature flags, system properties, singletons) in teardown that runs even on failure. *A hermetic per-test sandbox (fresh temp dir, transactional rollback, ephemeral container, monkeypatch context manager) torn down wholesale satisfies this without per-resource cleanup.*

```kotlin
// WRONG — shared mutable fixture + real clock + sleep-to-synchronize
companion object { val sharedCart = Cart() }          // bleeds state across tests

@Test fun addItem_thenCheckout_succeeds() {
    sharedCart.add(item)                              // next test sees this item
    service.checkout(sharedCart)
    Thread.sleep(500)                                 // flaky: hopes async settle finished
    assertThat(service.lastReceipt).isNotNull()
}

// RIGHT — fresh fixture, injected fake clock, awaited completion, pinned timezone where it matters
@Test fun checkout_withOneItem_producesReceipt() {
    val cart = Cart().apply { add(anItem()) }         // fresh per test
    val service = CheckoutService(clock = FakeClock(Instant.EPOCH))

    val receipt = service.checkout(cart).await()      // deterministic, no sleep

    assertThat(receipt.lineItems).hasSize(1)
}
```

## Q14.3 — Body Structure: Arrange / Act / Assert

Every test body MUST be three blocks in this order, separated by single blank lines:

| Block | Purpose | Notes |
|---|---|---|
| **Arrange** | Set up system, build inputs, prime test doubles | No assertions |
| **Act** | Single invocation of the behavior under test | Exactly one logical operation |
| **Assert** | Check observable outcomes | Targets the Act's effect |

- **MUST** have exactly one Act block — one logical invocation; multiple Acts MUST be split into separate tests.
- **MUST NOT** contain assertions before the Act — a precondition belongs in a shared helper or a separate test.
- **MUST** assert at most one logical outcome (several physical asserts that verify one cohesive outcome are fine; two independent behaviors in one body are not).
- **SHOULD** separate blocks by exactly one blank line, with no blank lines inside a block.
- **SHOULD** keep the body short enough to read without scrolling — a unit test over ~30 lines, or whose Arrange dwarfs Act and Assert, signals the unit does too much or the setup belongs in a factory (Q20). *An integration scenario wiring several real collaborators MAY be longer, provided Act and Assert remain singular and obvious.*
- **SHOULD** omit comment markers (`// Arrange`, `// Act`, `// Assert`) — the blank-line separation is the contract. *Keep the labels when a test is unavoidably long and the markers materially aid navigation.*

These MUST NOT appear in a test body: multiple Acts; assertions before the Act; conditional assertions (`if (x) assert...`) or loop assertions with no failure message; a `try { act(); fail() } catch (e) { /* empty */ }` swallow; a bare `act()` with the assertion in a `finally` or after an unconditional `return`.

```kotlin
@Test
fun fetchUser_withExpiredToken_throwsUnauthorized() {
    val expiredToken = AuthToken(value = "expired", expiresAt = Instant.EPOCH)
    val client = UserClient(httpClient = mockHttp, clock = fixedClock)
    every { mockHttp.get(any(), any()) } returns Response(401, body = "")

    val exception = assertThrows(UnauthorizedException::class.java) {
        client.fetchUser("user-123", expiredToken)
    }

    assertThat(exception.message).contains("user-123")
    verify(exactly = 1) { mockHttp.get(any(), any()) }
}
```

## Q14.4 — Assertion Quality

- **MUST** contain at least one assertion that can fail; a no-assertion test MUST NOT exist, and a "no exception thrown" test MUST make that explicit (an assert-does-not-throw call).
- **MUST** be specific — assert the actual expected value, exact size, or specific field, never merely `result != null`, `list.isNotEmpty()`, `response.ok`, `result.length > 0`, or `count >= 1` when the name promises a concrete value. *A weak assertion is acceptable only when the precise value is genuinely nondeterministic by design (a generated UUID, a current timestamp); then assert the property it must satisfy (format, range, uniqueness, monotonicity).*
- **MUST** pair a negative/absence assertion (`verify(never())`, `assertFalse`, "no exception") with a positive assertion proving the intended thing DID happen, so the test fails if the unit silently no-ops.
- **MUST** make expected values literals or independently derived, never computed by re-running the production logic under test (`assertEquals(formatter.format(x), result)` from the same `formatter` is a tautology).
- **MUST** compare floating-point and time with an explicit tolerance or a fixed/frozen reference, never naive equality on values subject to rounding or drift.
- **MUST**, on an error path, assert the specific error type AND a stable part of the error (code, message fragment, field) — not merely that "an error" occurred, and **MUST NOT** match on the full localized message string.
- **MUST** make interaction assertions check arguments and call count, not merely that a method was called; argument matchers MUST be as specific as the scenario allows (`any()` for a value the scenario actually constrains is a hole).
- **SHOULD** carry enough context to diagnose a failure without rerunning — a fluent matcher or explicit message over a bare boolean. *A single obvious equality assertion needs no custom message; the framework's diff is sufficient.*

```python
# WRONG — vacuous and tautological assertions
def test_render_invoice_returns_html():
    html = render_invoice(invoice)
    assert html is not None                       # passes for "" and for wrong HTML

def test_format_money_formats_correctly():
    expected = format_money(1999, "USD")          # re-runs the code under test
    assert format_money(1999, "USD") == expected  # always true, tests nothing

# RIGHT — specific value, independently derived expectation
def test_render_invoice_includes_total_line():
    html = render_invoice(an_invoice(total_cents=1999, currency="USD"))

    assert "<td class='total'>$19.99</td>" in html   # exact expected fragment
```

```typescript
// WRONG — interaction without arguments; error path with no type check;
// a negative assertion that passes even if the unit does nothing at all
expect(emailClient.send).toHaveBeenCalled()         // sent *something* — could be wrong
await expect(charge(card)).rejects.toThrow()         // any throw passes, even a typo
expect(auditLog.write).not.toHaveBeenCalled()        // also true if the whole flow no-ops

// RIGHT — argument + count asserted; specific error type and field; negative paired with a positive
expect(emailClient.send).toHaveBeenCalledTimes(1)
expect(emailClient.send).toHaveBeenCalledWith({ to: "a@b.com", template: "receipt" })
await expect(charge(declinedCard)).rejects.toMatchObject({
  name: "CardDeclinedError",
  code: "insufficient_funds",
})
expect(result.status).toBe("declined")              // proves the flow executed...
expect(auditLog.write).not.toHaveBeenCalled()       // ...and then asserts the absence
```

## Q14.5 — Edge-Case & Boundary Coverage

- **MUST**, for any function whose behavior changes across an input boundary, cover at minimum: empty/zero, single-element, a typical case, and each boundary value (min, max, just-below, just-above).
- **MUST** test the absent value of every nullable/optional parameter and the empty collection of every collection parameter.
- **MUST** exercise every documented error condition and every `throw`/`reject`/`Result.failure` branch in the modified code (mirrors Data Integrity Q13.6 and Error Handling: the fail-closed path MUST be tested).
- **MUST** test invalid, malformed, and adversarial inputs (wrong type, out-of-range, injection-shaped, truncated, oversized) for any code that parses, validates, or trusts external input, asserting it rejects them safely rather than corrupting state or crashing.
- **MUST** include the classic edge cases for known-tricky domains: text (empty, whitespace-only, leading/trailing whitespace, Unicode/multi-byte, combining characters, mixed scripts, very long, embedded control characters), numbers (zero, negative, overflow/underflow boundary, NaN/Infinity where representable, integer-width min/max), time (DST transition, leap day, the 23:59:59→00:00:00 day boundary, leap second only where the platform's time type can represent it, epoch, far-future, end-of-month, timezone boundary), collections (empty, one, duplicates, unsorted, null elements, very large), and concurrency (the same input arriving twice — idempotency — where the contract claims it; see Q14.6).
- **SHOULD** use parameterized/table-driven tests so adding a newly discovered edge case is one row, not one copy-pasted method. *Write separate named tests when each case needs materially different arrange/assert logic and a table would obscure rather than clarify.*

```typescript
// WRONG — one happy-path case; empty, single, and boundary inputs untested
it("splits a list into pages", () => {
  expect(paginate([1, 2, 3, 4, 5], 2)).toEqual([[1, 2], [3, 4], [5]])
})

// RIGHT — table-driven boundary coverage plus documented error and adversarial branches
it.each([
  { items: [],            size: 2, expected: [] },                  // empty
  { items: [1],           size: 2, expected: [[1]] },               // single
  { items: [1, 2, 3, 4],  size: 2, expected: [[1, 2], [3, 4]] },    // exact fit
  { items: [1, 2, 3],     size: 2, expected: [[1, 2], [3]] },       // remainder
])("paginate($items, $size) -> $expected", ({ items, size, expected }) => {
  expect(paginate(items, size)).toEqual(expected)
})

it("rejects a non-positive page size", () => {
  expect(() => paginate([1], 0)).toThrow(RangeError)                // documented error branch
})

it("rejects a negative page size without corrupting state", () => {
  expect(() => paginate([1, 2], -1)).toThrow(RangeError)            // adversarial / malformed
})
```

## Q14.6 — Async & Concurrency Correctness

- **MUST** settle every promise/future/coroutine it starts and have the runner observe that settlement; every awaitable assertion MUST be awaited (a forgotten `await`/`return` passes vacuously — the test asserts nothing).
- **MUST NOT** swallow unhandled rejections or background-task failures; tests **SHOULD** fail when a promise rejects without a handler or a background coroutine throws, and a suite config that ignores unhandled rejections MUST NOT be used to hide a real failure.
- **MUST** wait for an async condition with an explicit bounded await-until-condition (poll-and-assert, framework `waitFor`, awaited completion signal), never a fixed `sleep` — the timeout generous enough not to flake under load yet bounded so a hang fails fast.
- **MUST** exercise concurrency contracts (locks, idempotency, exactly-once, ordering) deterministically via a controlled scheduler, an injected barrier/latch, or a fake executor — not a hopeful parallel `for` loop. *A stress/soak test running many iterations to surface rare races is legitimate as a complement, but lives in the integration tier (Q20) and MUST NOT be the only coverage of a documented concurrency contract.*

```typescript
// WRONG — the assertion is never awaited; the test passes before it runs
it("rejects an expired token", () => {
  expect(fetchUser(expiredToken)).rejects.toThrow(UnauthorizedError)   // missing await/return
})

// RIGHT — the awaitable assertion is awaited; the runner observes the rejection
it("rejects an expired token", async () => {
  await expect(fetchUser(expiredToken)).rejects.toThrow(UnauthorizedError)
})

// RIGHT — wait for a condition with a bounded timeout, not a fixed sleep
it("flushes the buffer after the batch fills", async () => {
  const sink = new RecordingSink()
  const buffer = new BatchBuffer(sink, { size: 3 })

  buffer.add("a"); buffer.add("b"); buffer.add("c")

  await waitFor(() => expect(sink.flushed).toEqual(["a", "b", "c"]), { timeout: 1000 })
})
```

## Q15 — Test Doubles at the Boundary

- **MUST** substitute doubles only at an architectural boundary you own or wrap (a port, a repository interface, an HTTP client wrapper) — never a private method or a concrete internal collaborator.
- **MUST** match the real collaborator's contract: same return shapes, same thrown error types, same null/empty conventions, same latency-failure modes (timeout, partial result) where the scenario depends on them. **SHOULD** use contract/verified doubles (a shared contract test, a recorded interaction, a typed mock checked against the real interface) so a double cannot drift from a signature the real type no longer has.
- **MUST** avoid over-mocking — a test that is entirely mock setup and `verify` calls asserts a tautology; prefer a fake (a working in-memory implementation) for stateful collaborators and assert on observable state over interactions.
- **MUST NOT** let shared happy-path mock setup silently satisfy an error-path test; each test owns the double behavior its scenario requires.
- **MUST NOT** use type-only stubs or `as`/`!!`/unchecked casts to fabricate a double that omits fields the real type requires; construct a complete double or a fake instead. *A partial double is acceptable for a collaborator the scenario never calls, provided the test fails loudly (not silently returns a default) if an un-stubbed method is invoked.*
- **SHOULD** choose dummy / stub / fake / spy / mock by need: a stub for a canned return, a fake when behavior matters across calls, a spy/mock only when the interaction itself is the behavior under test. *A mocking framework's auto-stub is fine for collaborators irrelevant to the scenario, provided no assertion depends on them.*

```kotlin
// WRONG — mocks a concrete internal method; stub returns a shape the real gateway never sends
val taxGateway = mockk<TaxGateway>()
val service = spyk(OrderService(tax = taxGateway))
every { service.computeTax(any()) } returns Money.zero()       // mocking the unit's own logic
every { taxGateway.rate(any()) } returns TaxRate(negative = true) // real API never returns negative

// RIGHT — fake at the boundary, real logic exercised, contract-faithful
class FakeTaxGateway(
    private val rate: TaxRate,
    private val knownRegions: Set<Region>,
) : TaxGateway {
    // one method, the real interface's shape: throws for unknown regions at runtime
    override fun rate(region: Region): TaxRate =
        if (region in knownRegions) rate else throw UnknownRegionException(region)
}

@Test fun checkout_inTaxedRegion_addsTax() {
    val service = OrderService(tax = FakeTaxGateway(TaxRate.percent(8), setOf(Region.EU)))

    val total = service.checkout(cartOf(price = Money.dollars(100), region = Region.EU))

    assertThat(total).isEqualTo(Money.dollars(108))            // observable state, not interaction
}
```

## Q15.1 — No Unverified or Leaking Doubles

- **MUST** verify the expectations of a mock configured with strict expectations; **SHOULD** assert no unexpected interactions occurred (`verifyNoMoreInteractions` / `verifyNoUnverifiedInteractions`) where the contract is "these calls and no others," so a stray side effect (duplicate charge, stray notification) fails the test. *Omit the no-more-interactions assertion when the collaborator legitimately receives incidental calls the scenario does not constrain.*
- **SHOULD** remove stubbed responses that are never invoked, and **SHOULD** run frameworks that detect unnecessary stubbing in strict mode.
- **MUST** reset a process/module-level double (a global `jest.mock`, a static interceptor, a replaced singleton) between tests in failure-safe teardown (restates Q14.2 for doubles).

```typescript
// WRONG — primes a stub never exercised and never verified; nothing fails if send() vanishes
emailClient.send.mockResolvedValue({ id: "msg-1" })
const receipt = await checkout(cart)
expect(receipt.total).toBe(108)
// (send() may never have been called — the test still passes)

// RIGHT — verify the interaction and assert no stray side effects occurred
emailClient.send.mockResolvedValue({ id: "msg-1" })

const receipt = await checkout(cart)

expect(receipt.total).toBe(108)
expect(emailClient.send).toHaveBeenCalledTimes(1)
expect(paymentClient.charge).toHaveBeenCalledTimes(1)   // exactly one charge — no duplicate
```

## Q16 — Property-Based Testing

- **MUST** give pure functions (parsers, transformers, serializers, codec round-trips, comparators, normalizers) at least one property-based test asserting an invariant over generated inputs, in addition to example-based tests for known cases.
- **MUST** add a discovered failing input (a shrunk counterexample) as a permanent named regression test (see Q19).
- **MUST** be deterministically reproducible on failure — the framework reports the seed (or shrunk input) and a failing run replays from it (a non-replayable property test is itself flaky, Q14.2). *An explicitly time-boxed fuzzing job that logs its seed satisfies reproducibility without pinning a single seed.*
- **MUST** make generators cover the real input domain — including the empty, the maximal, and the structurally degenerate (deeply nested, duplicate-heavy, Unicode-heavy); a narrow generator gives false confidence and **SHOULD** be widened. *Constrain the generator when part of the domain is genuinely impossible by construction, and document the constraint at the generator.*

Common properties:
- **Roundtrip**: `decode(encode(x)) == x`
- **Idempotency**: `f(f(x)) == f(x)`
- **Commutativity**: `f(g(x)) == g(f(x))` where expected
- **Monotonicity**: `x <= y => f(x) <= f(y)` where expected
- **Invariant preservation**: `isValid(x) => isValid(transform(x))`
- **Oracle agreement**: `fast(x) == reference(x)` against a slow but obviously-correct implementation

```python
# RIGHT — roundtrip property plus an invariant, over generated inputs
from hypothesis import given, strategies as st

@given(st.text())
def test_encode_decode_roundtrips(s):
    assert decode(encode(s)) == s            # holds for every string, including "" and Unicode

@given(st.lists(st.integers()))
def test_sort_is_idempotent_and_preserves_length(xs):
    once = my_sort(xs)
    assert my_sort(once) == once             # idempotent
    assert len(once) == len(xs)              # no element dropped or duplicated
```

Tools: Hypothesis (Python), QuickCheck (Haskell), fast-check (TypeScript), kotest-property (Kotlin), jqwik (Java), SwiftCheck (Swift), proptest / quickcheck (Rust).

## Q17 — Mutation Testing

- **MUST** achieve an **85% mutation kill rate** on modified lines when a mutation tool is in scope for the language; a surviving mutant is a located test gap and MUST be closed by strengthening an assertion or adding a case, not by excluding the mutant.
- **MUST** include the operators that catch the defects this corpus cares about — at minimum conditional-boundary, negate-conditional, return-value, and removed-call where the tool supports them; a 100% kill rate from trivial operators only is a vanity number and MUST NOT be treated as satisfying this rule. *An operator the tool cannot apply to the language/idiom MAY be omitted, documented at the config.*
- **MUST NOT** silence a surviving mutant with a blanket equivalence annotation unless it is genuinely equivalent (semantically identical to the original), justified in a one-line comment at the suppression site.
- **SHOULD** scope mutation testing to changed code in pre-merge runs and run broadly on a schedule. *A project MAY raise this bar — but not lower it — in `.oracle/config.json`; a lower bar requires a documented rationale approved in review.*

Tools: Stryker (JS/TS), pitest (Java/Kotlin), mull (C/C++), mutmut / cosmic-ray (Python), cargo-mutants (Rust).

## Q18 — Coverage

- **MUST** achieve modified-line (diff) coverage ≥ **85%** as branch coverage, not merely statement coverage; statement-only coverage that leaves an `else` untested MUST NOT be counted as sufficient.
- **MUST** exercise both the true and false branch of every modified conditional, and every arm of a modified `switch`/`when`/`match` including default/else; **SHOULD** exercise independent clauses of a multi-clause boolean (condition/decision coverage) where the tool supports it. *A truly unreachable `else`/default that exists only to satisfy exhaustiveness MAY be excluded under the annotation rule below.*
- **MUST** enforce the coverage gate in the build (failing below threshold), verifiable in the build attestation; a gate set to skip-on-failure MUST NOT be used.
- **MUST** justify coverage exclusion annotations (`// istanbul ignore`, `# pragma: no cover`, `@Generated`) at the suppression site, and MUST NOT use them over hand-written branching logic — they are for generated code and truly unreachable defensive lines only. *A project MAY raise the threshold in `.oracle/config.json`; lowering it requires a documented, review-approved rationale.*

## Q19 — Regression Tests for Fixed Bugs

- **MUST** accompany every bug fix with a test that fails against the unfixed code and passes against the fix; the test MUST be written (or confirmed to fail) before the fix is applied.
- **MUST** encode the specific triggering condition (exact input, ordering, or state); **SHOULD** reference the originating issue in a comment. *Omit the reference when no tracked issue exists, but the test name and a one-line comment MUST still explain what defect it guards against.*
- **MUST NOT** be a duplicate of an existing happy-path test with a renamed method; target the coverage gap the bug exposed.
- **MUST NOT** be weakened or deleted to make an unrelated refactor pass; if the guarded behavior is intentionally changing, the change MUST update the assertion deliberately and the commit MUST state why.

```typescript
// RIGHT — reproduces the exact reported defect, named for the behavior it guards
// Guards against the empty-discount-code crash: blank codes were treated as a 100% discount.
it("treats a blank discount code as no discount, not full price off", () => {
  const total = applyDiscount(Money.dollars(50), /* code */ "   ")

  expect(total).toEqual(Money.dollars(50))   // pre-fix: returned Money.zero()
})
```

## Q20 — Test Maintainability

- **MUST NOT** merge a skipped/disabled/commented-out test (`@Ignore`, `.skip`, `xit`, `@Disabled`, `#[ignore]`, `it.todo`, `t.Skip`) without a tracked reason and an owner — a permanently skipped test is dead code and MUST be fixed or deleted; **MUST NOT** merge a `.only`/`fit`/`fdescribe` focus marker (it silently disables every other test in the file).
- **MUST NOT** contain branching or looping that itself needs testing — restructure conditional/looped assertions as parameterized cases (Q14.5). **SHOULD NOT** nest control flow more than one level. *A simple loop that builds a fixture (not one that contains assertions) is fine.*
- **SHOULD** name magic values whose meaning is not self-evident (an unexplained `42`, `"a@b.com"`, `Instant.ofEpochMilli(1623456789000L)`) for the boundary or condition they represent (`MAX_RETRIES`, `EXPIRED_AT`). *Obviously-illustrative literals in a table-driven case (`size: 2`) need no name; the column header carries the meaning.*
- **SHOULD** express shared setup as named factories/builders with sensible defaults and per-test overrides, not a giant `beforeEach`; a reader **MUST** be able to see in the test body every input that matters to its outcome.
- **MUST NOT** hide assertion logic behind a custom helper that obscures which value failed; **SHOULD** surface the offending input in the helper's failure message, or keep the assertion inline.
- **SHOULD** keep the full unit-test suite fast enough to run on every save (FIRST: Fast) and separate slow tests (real I/O, large fixtures) into an integration tier; the pyramid SHOULD hold — most coverage in fast unit tests, less in integration, least in slow end-to-end. *Integration and end-to-end tests are legitimately slower and live in their own tier with their own latency budget.*
- **MUST NOT** make assertions or fixtures depend on incidental, unstable details (map iteration order, full-object equality including timestamps, log line formatting, auto-generated ids, floating-point exactness) where the behavior under test does not constrain them — assert the relevant fields, sort before comparing unordered collections, freeze the clock, compare numbers within a tolerance.

```kotlin
// WRONG — disabled with no reason; full-object equality couples to an incidental timestamp
@Disabled
@Test fun computesSummary() {
    val summary = service.summarize(orders)
    assertThat(summary).isEqualTo(expectedSummary)   // fails when `generatedAt` drifts by 1ms
}

// RIGHT — active; asserts the fields the behavior constrains, ignores incidental ones
@Test fun summarize_withTwoOrders_sumsTotalsAndCounts() {
    val summary = service.summarize(listOf(orderOf(10), orderOf(15)))

    assertThat(summary.orderCount).isEqualTo(2)
    assertThat(summary.totalCents).isEqualTo(2500)   // ignores generatedAt by design
}
```

## Q21 — Snapshot & Golden-File Testing

- **MUST** review a snapshot/golden baseline as deliberately as production code when created or changed — inspect a regenerated snapshot line-by-line, never bulk-accept; regenerating with an update-all flag and committing without reading MUST NOT be done.
- **MUST** keep snapshots deterministic — no timestamps, random ids, absolute paths, hostnames, unordered-collection ordering, or environment-specific output; normalize/mask such fields before snapshotting (restates Q14.2/Q20 for snapshots).
- **SHOULD** keep a snapshot small and targeted at the output the test is about — a 2,000-line whole-DOM/whole-response snapshot verifies nothing a reviewer can reason about; prefer the specific fragment or a small structural projection. *A deliberately whole-artifact golden file (generated-code or serialized-schema fixture whose entire content is the contract) is legitimate, provided its determinism is enforced.*

```typescript
// WRONG — snapshots the whole response including a live timestamp and a random id;
// the next run churns the baseline, and update-all enshrines whatever came out
expect(renderReceipt(order)).toMatchSnapshot()
// baseline contains: { id: "f3a9...", generatedAt: "2026-06-13T22:01:07.413Z", total: "$19.99" }

// RIGHT — normalize volatile fields, snapshot a stable projection, assert the field under test explicitly
expect(renderReceipt(order)).toMatchSnapshot({
  id: expect.any(String),                  // identity not under test here
  generatedAt: expect.any(String),         // masked: volatile
})
expect(renderReceipt(order).total).toBe("$19.99")
```

## Q22 — Test Data & Fixture Safety

- **MUST NOT** contain real secrets, credentials, API keys, tokens, or live personal data in fixtures — use obviously-synthetic values (`"test-token"`, `user@example.com`, RFC-reserved ranges, documented test card numbers); a real secret committed in a fixture is a security incident.
- **MUST NOT** connect to, mutate, or depend on a shared or production resource (a real account, a shared staging database, a production endpoint, a colleague's machine); resource targets MUST come from injected, test-scoped configuration defaulting to a local/ephemeral instance.
- **MUST NOT** exfiltrate or print sensitive data on failure — failure output **SHOULD** surface only the fields under test, redacting or omitting sensitive ones. *Synthetic, non-sensitive fixture data MAY be printed freely; the rule binds only to real or realistic-sensitive values.*

```python
# WRONG — embeds a real-looking secret and points the test at a shared resource
def test_client_authenticates():
    client = ApiClient(
        token="sk_live_EXAMPLE_live_shaped_key",         # a real-shaped live key
        base_url="https://api.prod.internal/v1",          # shared/production endpoint
    )
    assert client.authenticate().ok

# RIGHT — synthetic credential, injected test-scoped endpoint defaulting to a local fake
def test_client_authenticates_against_local_fake():
    client = ApiClient(
        token="sk_test_FAKE_DO_NOT_USE",                  # obviously synthetic
        base_url=local_fake_server.url,                   # ephemeral, test-scoped
    )

    assert client.authenticate().ok
```

## Self-Review Checklist

| # | Check | Required | Rule |
|---|---|---|---|
| 1 | Test names state scenario AND expected outcome (`test_<method>_<scenario>_<expected>` or equivalent idiom) | Required | Q14.1 |
| 2 | No vague names (`works`, `happyPath`, `success`, bare ticket number); name stays true to what the body asserts | Required | Q14.1 |
| 3 | No order-implying sequenced names (`step1`, `first`/`second`); each name stands alone | Required | Q14.1 |
| 4 | Each test is independent — passes in isolation, in reverse order, in parallel, and under randomized order | Required | Q14.2 |
| 5 | No shared mutable fixtures; each test builds its own inputs via factory/builder | Required | Q14.2 |
| 6 | No real network/file/db/clock/randomness/env in unit tests; boundaries substituted | Required | Q14.2 |
| 7 | Locale/timezone/charset pinned where the assertion depends on them | Required | Q14.2 |
| 8 | Time and randomness controlled (fake clock / seeded RNG); no sleep-to-synchronize | Required | Q14.2 |
| 9 | Flaky tests root-caused and fixed/quarantined, never papered over with retries/sleeps/wider timeouts | Required | Q14.2 |
| 10 | External and global state created/mutated by a test restored in failure-safe teardown | Required | Q14.2 |
| 11 | Body structured as Arrange / Act / Assert | Required | Q14.3 |
| 12 | Exactly one Act block per test | Required | Q14.3 |
| 13 | No assertions before the Act | Required | Q14.3 |
| 14 | At most one logical outcome asserted per test | Required | Q14.3 |
| 15 | One blank line between AAA blocks; no blank lines inside a block | Required | Q14.3 |
| 16 | Body short and readable; oversized Arrange extracted to a factory | Required | Q14.3 |
| 17 | No conditional/looped assertions; no empty-catch swallow; no assertion skippable by control flow | Required | Q14.3 |
| 18 | Every test has at least one assertion that can fail; no no-assertion tests | Required | Q14.4 |
| 19 | Assertions are specific (exact value/size/field), not merely non-null/non-empty/`> 0` | Required | Q14.4 |
| 20 | Negative/absence assertions paired with a positive assertion proving the flow ran | Required | Q14.4 |
| 21 | Expected values are literals or independently derived, never re-run production logic | Required | Q14.4 |
| 22 | Float/time comparisons use tolerance or a frozen reference, not naive equality | Required | Q14.4 |
| 23 | Error-path tests assert specific error type AND a stable message-code/field (not full localized text) | Required | Q14.4 |
| 24 | Interaction assertions check arguments and call count, not bare "was called"; matchers as specific as the scenario | Required | Q14.4 |
| 25 | Empty / single / typical / each-boundary cases covered for boundary-sensitive code | Required | Q14.5 |
| 26 | Each nullable/optional gets an absent-value test; each collection gets an empty test | Required | Q14.5 |
| 27 | Every documented error condition / throw branch in modified code is exercised | Required | Q14.5 |
| 28 | Malformed/adversarial inputs tested for parse/validate/trust code | Required | Q14.5 |
| 29 | Classic edge cases covered for text/number/time/collection/concurrency domains | Required | Q14.5 |
| 30 | Every awaitable assertion is awaited; async tests cannot pass vacuously | Required | Q14.6 |
| 31 | Unhandled rejections / background failures fail the test, not silently ignored | Required | Q14.6 |
| 32 | Async waits use bounded await-until-condition, never a fixed sleep | Required | Q14.6 |
| 33 | Concurrency contracts exercised deterministically (scheduler/barrier), not a hopeful parallel loop | Required | Q14.6 |
| 34 | Test doubles substituted only at an owned boundary, never internal/private methods | Required | Q15 |
| 35 | Double behavior matches the real collaborator's contract (shapes, errors, null/empty, failure modes) | Required | Q15 |
| 36 | No over-mocking tautologies; fakes preferred for stateful collaborators | Required | Q15 |
| 37 | No type-only/cast doubles missing required members; un-stubbed calls fail loudly | Required | Q15 |
| 38 | Primed mock expectations are verified; no unexpected-interaction or duplicate side effects | Required | Q15.1 |
| 39 | Unused stubs removed; global/module doubles reset between tests | Required | Q15.1 |
| 40 | Pure functions have property-based tests asserting an invariant over generated inputs | Required | Q16 |
| 41 | Shrunk counterexamples added as permanent named regression tests | Required | Q16 |
| 42 | Property-test failures are reproducible from a reported seed/shrunk input | Required | Q16 |
| 43 | Generators cover the real domain (empty/maximal/degenerate/Unicode); constraints documented | Required | Q16 |
| 44 | Modified lines achieve ≥ 85% mutation kill rate; survivors closed, not silenced | Required | Q17 |
| 45 | Mutation run includes boundary/negate/return/removed-call operators, not just trivial ones | Required | Q17 |
| 46 | Equivalent-mutant suppressions justified in a one-line comment | Required | Q17 |
| 47 | Modified-line branch coverage ≥ 85% | Required | Q18 |
| 48 | Both branches of every modified conditional, and every switch/default arm, exercised | Required | Q18 |
| 49 | Coverage gate enforced in build (not skipped) and verifiable in attestation | Required | Q18 |
| 50 | Coverage exclusions justified and not over hand-written branching logic | Required | Q18 |
| 51 | Every bug fix has a test that fails pre-fix and passes post-fix | Required | Q19 |
| 52 | Regression test encodes the exact triggering condition and explains the defect | Required | Q19 |
| 53 | Regression test is not a renamed happy-path duplicate; not silently weakened by an unrelated refactor | Required | Q19 |
| 54 | No merged skipped/disabled tests without a tracked reason and owner; no `.only`/focus markers | Required | Q20 |
| 55 | No branching/looping-with-assertions and no >1-level nesting inside test logic; use parameterized cases | Required | Q20 |
| 56 | Unexplained magic values named; inputs that matter are visible in the test body, not hidden setup | Required | Q20 |
| 57 | Custom assertion helpers surface the offending value; suite respects the test pyramid (fast unit tier) | Required | Q20 |
| 58 | Assertions/fixtures do not depend on incidental unstable details (order, timestamps, ids, float exactness) | Required | Q20 |
| 59 | Snapshot/golden baselines reviewed line-by-line on change, never bulk-accepted | Required | Q21 |
| 60 | Snapshots normalize volatile fields (timestamps, ids, paths, ordering) and stay small/targeted | Required | Q21 |
| 61 | Fixtures contain no real secrets/credentials/PII; only obviously-synthetic values | Required | Q22 |
| 62 | Tests target injected test-scoped/ephemeral resources, never shared/production endpoints | Required | Q22 |
| 63 | Failure output does not print real/sensitive data; surfaces only fields under test | Required | Q22 |
