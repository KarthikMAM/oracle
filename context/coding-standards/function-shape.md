# Function Shape

Functions are the units of cognition: their size, nesting, and branching decide whether a defect is obvious on the screen or buried three levels deep where no reviewer will catch it. **Posture (per `context/principles.md` → Simplicity):** a function reads straight down — guard clauses and early returns at the top, the happy path flat below, one level of abstraction, no cleverness the next reader must decode. Safety and correctness items are absolute (MUST) and BLOCK-severity for the clean-code reviewer lens; taste and maintainability items are SHOULD and carry a documented escape hatch so they never fight a legitimate edge case.

## Anchors

- **SRP** (Robert C. Martin) — one reason to change per function
- **Long Method smell** (Martin Fowler, *Refactoring*, 1999) — extract until each function does one thing
- **Cyclomatic Complexity** (McCabe, 1976) — independent paths are the count of tests you owe
- **Cognitive Complexity** (G. Ann Campbell / Sonar, 2018) — nesting compounds the cost of understanding
- **POLA** (Saltzer & Schroeder, 1975) — a function does what its name and signature promise, nothing hidden
- **Command-Query Separation** (Bertrand Meyer, *Object-Oriented Software Construction*, 1988) — a function either changes state or answers a question, never both
- **Single Level of Abstraction** (Kent Beck / Martin Fowler) — every statement in a function sits at one altitude

## Q3.1 — Size and Nesting

- **SHOULD** keep each function to at most **20 lines of logic** (excluding doc comments, blank lines, imports, closing-brace-only lines). *Orchestrator escape hatch: a linear sequence of calls to well-named helpers — no loops, no conditionals beyond top-of-body guards, no nested logic — MAY run to 40 lines, and SHOULD carry a one-line comment marking it an orchestrator.*
- **SHOULD** keep at most **1 level of nesting**; extract anything deeper into a named helper. *A single-statement `switch`/`when`/`match` arm and a single-expression closure passed to `map`/`filter`/`forEach` do NOT count as nesting; genuinely irreducible nesting (a documented matrix walk) MAY use 2 levels with an inline justification.*
- **SHOULD** place guard clauses and early returns at the top so the happy path falls through below, never wrapped in the `if`. *A resource-acquisition block that must wrap the happy path (`try`-with-resources / `use` / `defer`, a lock or transaction span) MAY nest it.*
- **SHOULD** keep a small, bounded `return` count: top guard clauses plus **one** happy-path return; extract scattered nested returns. *A flat exhaustive `switch`/`when`/`match` MAY return once per arm.*
- **MUST NOT** use force unwraps, force casts, or force tries: `!`, `as!`, `try!`, implicitly-unwrapped optionals (`var x: T!`) in Swift; `!!` in Kotlin; unchecked/unsafe casts in Java/Kotlin; non-null assertion `!`, `as any`, double-cast `as unknown as T` in TypeScript; `.unwrap()`, `.expect()`, `panic!` on recoverable paths in Rust; a bare `interface{}`/`any` type assertion without the two-result `, ok` form in Go. *No escape hatch outside test fixtures.*
- **MUST NOT** use compiler/linter silencing annotations: `@Suppress`, `@SuppressWarnings`, `@ts-ignore`, `@ts-expect-error`, `@ts-nocheck`, `// eslint-disable*`, `// noinspection`, `// NOLINT`, `# type: ignore`, `# noqa`, `#[allow(...)]`, `//nolint`. *A single line-scoped suppression for a documented false positive in a third-party type stub or generated code MAY be used with an inline comment naming the tool, the rule, and why the warning is wrong; file-level or unscoped suppression MUST NOT.*

### Example: Nesting Extraction

```kotlin
// WRONG — nested if inside for (2 nesting levels)
fun processOrders(orders: List<Order>) {
    for (order in orders) {
        if (order.isValid) {
            ship(order)
        }
    }
}

// RIGHT — extract into named helper; body reads flat
fun processOrders(orders: List<Order>) {
    orders.filter(::isShippable).forEach(::ship)
}

private fun isShippable(order: Order): Boolean = order.isValid
```

### Example: Bounded Returns

```typescript
// WRONG — a return buried inside a loop inside an if; exits not visible at top
function firstBadRow(rows: Row[], limit: number): Row | undefined {
  if (rows.length > 0) {
    for (const row of rows) {
      if (row.score > limit) {
        return row
      }
    }
  }
  return undefined
}

// RIGHT — guard at top, single happy-path expression; one obvious exit
function firstBadRow(rows: Row[], limit: number): Row | undefined {
  if (rows.length === 0) return undefined

  return rows.find((row) => row.score > limit)
}
```

## Q3.2 — Complexity Limits

- **SHOULD NOT** let cyclomatic complexity exceed **5** per function. Count +1 for each independent branch point: `if`, `else if`, ternary, `&&`, `||`, `??`/null-coalescing, a loop construct (`for`/`while`/`do-while` and the loop-bearing higher-order calls `forEach`/`map`/`filter`/`reduce`/`some`/`every`/`find` — count the call once, plus any branching inside its closure), `catch`, each non-default `case`/`when`/`match` arm, and each `?` early-return in Rust. (Optional-chaining `?.` is null-guard sugar, not an independent path under McCabe — do not count it.)
- **SHOULD NOT** let cognitive complexity exceed **7** per function. Each control structure adds 1 plus current nesting depth; a `break`/`continue`/early `return` to a label adds 1; a sequence of mixed `&&`/`||` in one condition adds 1 per operator-kind switch. *Both limits: a function that is inherently a flat dispatch — parser token table, protocol state-machine transition, exhaustive `when`/`switch` over a sealed/enum type mapping each case to one expression — MAY exceed both counts; it MUST stay flat, MUST be covered by a test per arm, and MUST carry a one-line comment naming the exemption.*
- **SHOULD NOT** use ternary operators (Swift/TS/Java `?:`; Python `a if c else b`); extract into a named function or value. *A single, non-nested ternary for presentation/formatting at a leaf MAY be used; nested ternaries MUST NOT be used under any circumstance.*
- **SHOULD NOT** use `else` blocks for `if`-conditionals; prefer early-return guards, and convert `else if` chains of 3+ arms to a `switch`/`when`/`match` or lookup table. *A short chain of unrelated range tests on the same scalar (a histogram-bucket assignment) MAY stay as `else if`, each arm a single statement.*
- Carve-out: the `else`/`default` arm of an *exhaustive* `switch`/`when`/`match` is NOT a prohibited `else` — it is the required exhaustiveness arm. Kotlin `when` on `sealed` or `enum` subjects **MUST** omit `else ->` so the compiler enforces exhaustiveness and a new variant becomes a compile error rather than a silent fall-through.
- Carve-out: a single-expression value assignment `val x = if (cond) a else b` (Kotlin) or `let x = if c { a } else { b }` (Rust) is NOT a prohibited `else` block. **SHOULD** keep both branches single expressions; extract the moment a branch grows a statement.
- **SHOULD** extract deeply negated or compound conditions (`if (!a && !(b || c))`) into a positively-named predicate. *A single negation of a well-named predicate (`if (!isReady)`) is fine.*

### Example: Guard-Clause Pattern

```swift
// WRONG — else block
func fetchUser(id: String) -> User? {
    if let cached = cache[id] {
        return cached
    } else {
        return loadFromNetwork(id)
    }
}

// RIGHT — early-return guard, no else
func fetchUser(id: String) -> User? {
    if let cached = cache[id] { return cached }

    return loadFromNetwork(id)
}
```

### Example: Replace Nested Ternary with Named Decision

```typescript
// WRONG — nested ternary; the decision is unreadable and untestable
function tier(spend: number): string {
  return spend > 1000 ? "gold" : spend > 100 ? "silver" : "bronze"
}

// RIGHT — named, ordered, exhaustively testable
function tier(spend: Money): Tier {
  if (spend.greaterThan(GOLD_THRESHOLD)) return Tier.Gold
  if (spend.greaterThan(SILVER_THRESHOLD)) return Tier.Silver

  return Tier.Bronze
}
```

> A function's name is its contract surface; the naming taxonomy is owned by **naming.md (Q3.3)** and composes with the command-query rule in Q3.6 below. This file does not restate it.

## Q3.4 — Parameters

- **SHOULD** take at most **4 parameters**; beyond that, extract a parameter object / options type. *A framework-required signature (a callback whose shape you do not own, a platform lifecycle method) MAY exceed 4; document it inline.*
- **MUST NOT** let boolean (flag) parameters appear at call sites where the meaning is ambiguous (`render(true, false)`); use an enum or named-options type, or split into two intention-revealing functions.
- **MUST** disambiguate two or more **adjacent parameters of the same type** the caller could transpose — wrap each in a distinct domain type, use named/labeled arguments, or reorder so a transposition fails to compile.
- **MUST NOT** use output parameters (mutating a passed-in reference to return a result); return the result, or a tuple/data type for several outputs. *A profiled performance-critical path filling a caller-owned buffer (a `Span<T>`/`ByteBuffer` write, a Rust `&mut [u8]` sink) MAY take an out-buffer when the signature advertises the fill.*
- **MUST NOT** mutate its own parameters in place when the caller can still observe the argument (reassigning a local copy is fine; mutating a passed collection or object is not). *An explicitly-named in-place mutator (`sortInPlace`, a `&mut` Rust receiver, an `inout` Swift parameter) whose signature advertises the mutation.*
- **SHOULD** make default parameter values represent the most common usage and document them. *A self-evident, type-enforced default (an empty immutable collection, a `false` opt-in flag) MAY omit prose.* **MUST NOT** use mutable default arguments (Python `def f(items=[])`, shared default objects in TS/JS) — default to `None`/`undefined` and construct inside; no escape hatch.

### Example: Flag Parameter and Same-Type Transposition

```python
# WRONG — positional bool is unreadable; mutable default shared across calls
def export_report(line_item, include_totals=True, rows=[]):
    rows.append(line_item)     # accumulates across every call — classic shared-default bug
    return render(rows, include_totals)

# RIGHT — explicit Totals type replaces the opaque flag; safe default constructed inside
def export_report(line_item: LineItem, totals: Totals, rows: list | None = None) -> Report:
    if rows is None:
        rows = []
    rows.append(line_item)
    return render(rows, totals)
```

## Q3.5 — No Trivial Wrappers

- **SHOULD** inline a private function that only forwards its arguments unchanged to another function. *The wrapper stays if it satisfies an interface/protocol, narrows visibility, pins a default, adapts one type to another, or gives a stable seam for testing.*

## Q3.6 — Command-Query Separation and Side-Effect Honesty

- **SHOULD** be either a **command** (changes observable state, returns nothing meaningful) or a **query** (returns a value, changes nothing observable), not both. *Well-known idioms that must return a value to be usable — `pop()`, `getOrCreate`, `compareAndSet`, an iterator's `next()`, a builder's chaining `return this` — are sanctioned commands-with-results; their names advertise the dual nature.*
- **MUST NOT** give a query side effects a caller cannot see in the signature: no hidden cache writes, no lazy mutation, no logging-with-state-change, no I/O behind a name that promises a pure computation. If a "read" must populate a cache, name it so (see naming.md Q3.3) so the effect is in the contract.
- **SHOULD** push side effects (I/O, global/shared-state mutation, time, randomness) to the edges; core decision logic depends only on its inputs. *Thin I/O adapters and the orchestrator (Q3.1) are allowed to be effectful by nature.*

### Example: Query With Hidden Mutation

```typescript
// WRONG — a "query" mutates: reading the unread count marks everything read
function unreadCount(inbox: Inbox): number {
  const count = inbox.messages.filter((m) => !m.read).length
  inbox.messages.forEach((m) => (m.read = true))   // hidden command inside a query
  return count
}

// RIGHT — query is pure; the command is separate and named for its effect
function unreadCount(inbox: Inbox): number {
  return inbox.messages.filter((m) => !m.read).length
}

function markAllRead(inbox: Inbox): void {
  inbox.messages.forEach((m) => (m.read = true))
}
```

## Q3.7 — Exhaustive Branch and Edge-Case Handling

- **MUST** handle every case explicitly when branching on a closed set (an enum, a sealed type, a status code family); a catch-all `default`/`else` MUST NOT absorb future variants — use exhaustiveness checking (Kotlin `when` without `else`, Swift `switch` over an enum, TypeScript `never`-assertion default, Rust `match`) so a new variant is a compile error.
- **MUST** behave correctly on the **empty** and **single-element** collection without a special-case branch where the general case already covers them; where they genuinely differ (a "join with separator", a "first vs rest" fold), each MUST be handled and tested.
- **MUST** account for boundary values in numeric and indexing logic: minimum, maximum, zero, the off-by-one at `length`/`length - 1`, and arithmetic that can overflow or divide by zero; a function that indexes or slices MUST guard the bound rather than trust the caller. *The only relief is a precondition the type system already proves (a non-empty-collection type, an unsigned index).*
- **MUST** handle an optional/nullable input's absent case at the top of the function (a guard clause), not deferred to a force-unwrap deeper down (Q3.1).

### Example: Exhaustive Closed-Set Handling

```kotlin
// WRONG — else absorbs unknown states; adding PARTIAL_REFUND compiles and silently does nothing
fun describe(status: PaymentStatus): String = when (status) {
    PaymentStatus.PAID -> "paid"
    PaymentStatus.FAILED -> "failed"
    else -> "unknown"
}

// RIGHT — no else on a sealed/enum subject; a new variant is a compile error
fun describe(status: PaymentStatus): String = when (status) {
    PaymentStatus.PAID -> "paid"
    PaymentStatus.PENDING -> "pending"
    PaymentStatus.FAILED -> "failed"
}
```

### Example: Empty and Boundary Cases

```typescript
// WRONG — seedless reduce throws on empty input; and returning 0 for empty is an
// overloaded value a caller can't tell apart from a real average of 0 ([0], [-1,1]).
function average(values: number[]): number {
  return values.reduce((a, b) => a + b) / values.length   // empty => reduce on [] throws (no seed)
}

// RIGHT — empty is a distinct, unmistakable result; non-empty path is seeded and total
// `undefined` (not 0) so the caller can tell "no data" from a real average of 0 — no
// overloaded sentinel (error-handling.md Q7.6). A Result/Optional or a non-empty input
// type would satisfy the same contract.
function average(values: number[]): number | undefined {
  if (values.length === 0) return undefined

  const sum = values.reduce((a, b) => a + b, 0)
  return sum / values.length
}
```

## Self-Review Checklist

| # | Check | Required | Rule |
|---|---|---|---|
| 1 | Functions ≤20 lines of logic (orchestrator escape hatch ≤40, commented) | Required | Q3.1 |
| 2 | Max 1 nesting level inside the body (flat-arm / single-expr-closure exempt) | Required | Q3.1 |
| 3 | Guard clauses at top; happy path falls through; bounded return count | Required | Q3.1 |
| 4 | No force unwraps / casts / tries / implicitly-unwrapped optionals (all languages) | Required | Q3.1 |
| 5 | No `!!` (Kotlin), `as any` / `as unknown as` (TS), `.unwrap()`/`.expect()`/`panic!` on recoverable paths (Rust) | Required | Q3.1 |
| 6 | No compiler/linter silencing annotations (line-scoped documented stub exception only) | Required | Q3.1 |
| 7 | Cyclomatic complexity ≤5 per function (dispatch-table escape hatch, commented + tested) | Required | Q3.2 |
| 8 | Cognitive complexity ≤7 per function (flat-dispatch escape hatch, commented + tested) | Required | Q3.2 |
| 9 | No ternary operators (single leaf-presentation ternary escape hatch; never nested) | Required | Q3.2 |
| 10 | No `else` blocks; 3+ `else if` arms converted to switch/lookup (exhaustive default-arm exempt) | Required | Q3.2 |
| 11 | Kotlin `when`/Rust `let`-if value-assignment with single-expression branches exempt | Required | Q3.2 |
| 12 | Compound/negated conditions extracted to positively-named predicates | Required | Q3.2 |
| 13 | ≤4 parameters per function (framework-signature escape hatch documented) | Required | Q3.4 |
| 14 | Boolean/flag parameters use enum/named-options or split function at ambiguous call sites | Required | Q3.4 |
| 15 | Adjacent same-type parameters disambiguated (domain types / named args / reorder) | Required | Q3.4 |
| 16 | No output parameters; no caller-observable in-place argument mutation (named mutators exempt) | Required | Q3.4 |
| 17 | Default values documented; no mutable default arguments | Required | Q3.4 |
| 18 | Trivial forwarding wrappers inlined (interface/visibility/adapter/test-seam exempt) | Required | Q3.5 |
| 19 | Functions are command OR query, not both (sanctioned dual-nature idioms exempt) | Required | Q3.6 |
| 20 | Queries have no hidden side effects; effects pushed to edges, core logic pure | Required | Q3.6 |
| 21 | Closed-set branches handled exhaustively; no catch-all default absorbing future variants | Required | Q3.7 |
| 22 | Empty and single-element collection cases correct and tested | Required | Q3.7 |
| 23 | Boundary/overflow/divide-by-zero/off-by-one and absent-optional cases guarded | Required | Q3.7 |
