# Formatting — Visual Layout Rules

Visual layout rules — when a signature wraps, how blocks are spaced, where blank lines belong, how a diff reads. These rules survive the formatter because they encode judgment calls automated formatters cannot make: a misread signature, a buried `else`, or a reformat-laden diff hides defects from the human reviewer who is the last line of defense. Most are taste calls (SHOULD); a few guard correctness and stay absolute (MUST).

## Anchors

- **Code as Prose** (Knuth, *Literate Programming*, 1984) — source is written for a person first, the machine second
- **Stepdown Rule** (Martin, *Clean Code*, 2008) — code reads top-to-bottom, each function one level of abstraction below the one above
- **Newspaper Code** (Martin, 2008) — most important detail first, supporting detail descending
- **Optimize for Reading** (Hunt & Thomas, *The Pragmatic Programmer*, 1999) — code is read an order of magnitude more often than written
- **Functional Core, Imperative Shell** (Bernhardt, 2012) — pure computation grouped apart from side effects
- **Command-Query Separation** (Meyer, *OOSC*, 1988) — a block either asks or acts, never silently both
- **AAA** (Beck / xUnit) and **Given-When-Then** (North, BDD, 2006) — test bodies have three named phases
- **Diff Minimality** (Git community lore) — a reviewer's attention is finite; every line of noise spends it and hides the line that matters

## F1 — Signature Layout

- **SHOULD** keep a signature on one line when ALL hold: fits the per-language limit (Swift 120, Kotlin/Java 120, TypeScript 100, Python 88, Go ≤ 120, Rust 100); **3 or fewer** parameters; no parameter has a complex type (closure/lambda, generic with 2+ type params, tuple, union/sum type with 2+ members) or a default value; the return type is not a multi-clause generic that pushes past the limit.
- **SHOULD** wrap one parameter per line (indented one level, trailing comma where supported, opening `(` on the signature line, closing `)` aligned with the declaration start) when any condition above fails.
- **SHOULD** judge call sites independently of their declaration — a wrapped declaration does not force a wrapped call, and vice versa.
- **SHOULD NOT** use ragged layout (some params on the signature line, the rest wrapped) — a signature is wholly single-line or wholly one-param-per-line. *Escape hatch: a signature emitted by a generator or a formatter the author does not control MAY use whatever layout the tool produces; hand-written signatures conform.*

```swift
// WRONG — unnecessary wrap; 3 params, fits on one line
private static func validate(
    _ version: String,
    id: Any,
    state: RuntimeState
) -> Response? {

// RIGHT — keep on one line
private static func validate(_ version: String, id: Any, state: RuntimeState) -> Response? {
```

## F2 — Method Chains

- **SHOULD** break every call onto its own line (each leading with `.`) when a chain exceeds **2 calls** OR the line exceeds the limit; a chain of **≤2 calls** SHOULD stay on one line when it fits.
- **SHOULD NOT** mix styles in one chain (partially-broken). *Escape hatch: a chain emitted by a formatter the author does not control MAY use whatever layout the tool produces.*
- **MUST** hoist intermediate results that are reused, branched on, or logged into a named binding rather than burying them mid-chain.

```kotlin
// WRONG — partially broken; reviewer cannot diff a single stage cleanly
val names = users.filter { it.isActive }.map { it.name }
    .sorted().distinct()

// RIGHT — one stage per line
val names = users
    .filter { it.isActive }
    .map { it.name }
    .sorted()
    .distinct()
```

## F3 — Multi-line Single-Expression Functions

- **MUST** use an explicit `return` with a block body (not `=`/arrow expression syntax: Kotlin `=`, Swift implicit-return closures, Scala/TS arrow bodies) when a single-expression function body does not fit on the signature line.
- **MAY** keep expression syntax when the body DOES fit on the signature line — this rule fires only when the body wraps. *Escape hatch: a language idiom where the wrapped expression body is the established, unambiguous convention — a SwiftUI `some View` `body` whose single expression spans lines, a Scala for-comprehension result — MAY keep expression syntax when the return is structurally obvious.*

```typescript
// WRONG — wrapped implicit-return arrow; a later added line silently changes what is returned
const summarize = (o: Order) =>
    o.items.reduce((sum, i) => sum + i.price, 0)

// RIGHT — block body with explicit return once it wraps
const summarize = (o: Order): number => {
    return o.items.reduce((sum, i) => sum + i.price, 0)
}
```

## F4 — Argument Layout

- **SHOULD** wrap a call with **3 or more arguments** that exceeds the line limit to one argument per line (trailing comma where supported), and multi-line argument lists SHOULD NOT be ragged. *Escape hatch: a call emitted by a generator or formatter the author does not control MAY use whatever layout the tool produces.*
- **SHOULD** name an unused parameter to signal intent rather than leave it bare. The mechanism is per-language: a closure/lambda parameter or a pattern/destructuring binding that is unused takes the bare discard token (`_` for a Swift/Kotlin closure param, a Rust binding, Go's blank identifier `_`, a Python tuple-unpack slot); a Go function parameter that is unused takes the bare blank identifier `_`. An unused *regular named* function parameter that the language will not let you discard (Kotlin — `fun f(_: Int)` does not compile; Swift — `_` is the call-site label, not a discard, so `func f(_ x: Int)` still binds `x`) keeps a meaningful name, or use a `_`-prefixed name (`_unused`) in Java/TypeScript where a linter recognizes the convention; remove the parameter entirely when the signature is yours to change.
- **SHOULD** use argument labels / named arguments for positional arguments whose meaning is not obvious at the call site (bare booleans, bare numeric literals, two same-typed strings in a row). *Escape hatch: a single self-evident argument, or a language without named arguments (Java, Go) where an options object would be heavier than the readability it buys, MAY pass positionally.*

```swift
// WRONG — bare booleans at the call site; transposable without the compiler noticing
view.resize(true, false)

// RIGHT — argument labels document each value
view.resize(animated: true, preservingRatio: false)
```

## F5 — Vertical Spacing Inside Function Bodies

A well-shaped function body is a sequence of named logical blocks in this order:

| # | Block | Purpose |
|---|---|---|
| 1 | **Guard** | Precondition checks, early returns, fail-fast validation |
| 2 | **Setup** | Local bindings, derived values, dependencies pulled into scope |
| 3 | **Transform** | Pure computation — the function's reason for existing |
| 4 | **Side-effect** | I/O, logging, persistence, telemetry, mutations |
| 5 | **Return** | The result |

- **SHOULD** separate each block with exactly **one** blank line; consecutive blank lines SHOULD NOT appear inside a body.
- **SHOULD NOT** put a blank line *inside* a block — that is the signal to extract a named helper (composes with Function-Shape Q3.1 size limits).
- **SHOULD NOT** put a blank line after the opening brace or before the closing brace. A `return` is its own block (#5) and carries the one-blank-line separation per the rule above — the only case where a blank line before `return` is wrong is when the return is *not* a separate block (it is the sole or concluding statement of the same logical block, e.g. a one-line `Transform` whose value is returned directly); there, no blank line goes between the computation and its `return`.
- **SHOULD NOT** order blocks out of sequence; Side-effect-then-Transform is a Command-Query smell and SHOULD be reordered or split. *Escape hatch: an interleaving genuinely demanded by the algorithm (e.g. a streaming loop that emits as it computes) MAY deviate, carrying a one-line comment naming why the phases interleave.*

```kotlin
// WRONG — wall of text
fun processOrder(order: Order): Receipt {
    val validated = validator.validate(order)
    if (!validated.isValid) { return Receipt.invalid(validated.errors) }
    val priced = pricer.applyPricing(validated)
    val taxed = taxCalculator.apply(priced)
    logger.info("Order processed", mapOf("orderId" to order.id))
    return Receipt(order = taxed)
}

// RIGHT — blocks separated by exactly one blank line
fun processOrder(order: Order): Receipt {
    val validated = validator.validate(order)

    if (!validated.isValid) { return Receipt.invalid(validated.errors) }

    val priced = pricer.applyPricing(validated)
    val taxed = taxCalculator.apply(priced)

    logger.info("Order processed", mapOf("orderId" to order.id))

    return Receipt(order = taxed)
}
```

### Test Code — AAA / Given-When-Then

Three blocks, exactly one blank line between each:

| # | Block | Purpose |
|---|---|---|
| 1 | **Arrange** | Set up the system under test, build inputs, prime mocks |
| 2 | **Act** | Invoke the single behavior under test |
| 3 | **Assert** | Check observable outcomes |

- **MUST** have exactly one Act block per test.
- **MUST NOT** place any assertion before the Act block — an assertion in Arrange tests the fixture, not the behavior.
- **SHOULD** separate the three phases with blank lines so the behavior under test is visually unambiguous.

## F6 — Comment Placement

- Inline-comment *placement* (above the code, not end-of-line) is owned by the Documentation domain (documentation.md Q2.2 — a **MUST**, with its end-of-line exception for a short unit/enum annotation on a data declaration); this file does not restate it. The formatting-specific corollary: when an exception sanctions an end-of-line annotation, it MAY sit there even where moving it above would break a table's alignment, but it **MUST NOT** carry control-flow or rationale a reviewer needs in the code path.
- **SHOULD** use `// MARK: - Section Name` (Swift/Obj-C) or `// --- Section Name ---` (other languages) for section markers. *Escape hatch: a repo whose linter establishes a different canonical marker MAY follow it, applied consistently file-wide.*
- **SHOULD NOT** use `// region` / `// endregion` or IDE fold markers (`<editor-fold>`) — split the file instead.
- Commented-out code is owned by the Documentation domain (documentation.md Q2.2 — a **MUST** to remove it, with its sole exception for a commented line that is itself an example explicitly referenced by an adjacent comment, e.g. `// example invocation: foo(2, "ms")`); this file does not restate it. A genuinely needed disabled path is an explicit feature flag or a documented `if (false)`-with-rationale, not dead text.
- **SHOULD NOT** add a comment that merely restates the code (`// increment i`, `// return the result`) — comments explain *why*; the code shows *what* (composes with Documentation domain rules).

```swift
// WRONG — end-of-line comment carrying rationale a reader needs
let timeout = 5  // bumped from 2s because the upstream p99 regressed

// RIGHT — rationale on the line above
// Bumped from 2s because the upstream p99 regressed (see incident 4821).
let timeout = 5

// RIGHT — sanctioned end-of-line annotation on a literal table
let retryDelays = [1, 2, 4, 8]  // seconds, exponential backoff
```

## F7 — Diff Hygiene

- **SHOULD NOT** let reformat-only changes appear outside the hunks a change actually touches; a behavioral edit SHOULD NOT drag the surrounding file into a re-indent or re-wrap.
- **SHOULD NOT** mix reformatting with behavioral changes in one change set — ship the reformat as its own reviewable change, before or after.
- **SHOULD NOT** let whitespace-only churn (trailing-space removal, tab↔space flips, EOL-style flips) ride along in a behavioral change set.
- **SHOULD NOT** bundle import/dependency reordering into an unrelated change — stage only the imports the change actually adds or removes.
- **SHOULD** regenerate generated files (lockfiles, build outputs, snapshot fixtures) by their tool and commit them as a separate, clearly-labeled hunk — never hand-edited inside a behavioral diff.

```diff
# WRONG — one-line fix buried under a file-wide reindent
- 	if(x){doThing()}
+    if (x) {
+        doThing();
+    }
+    // ...200 lines of unrelated re-indentation in the same diff

# RIGHT — the behavioral fix alone; reformat shipped separately
- if (x) { doThing() }
+ if (x) { doThing(); logResult() }
```

## F8 — Indentation, Whitespace, and File Endings

- **MUST NOT** mix tabs and spaces for *indentation* within a file. Tool-produced alignment is exempt (mirroring the formatter-output exemption F7 grants): `gofmt` indents with tabs but column-aligns struct field tags, grouped const/var/import blocks, and line-end comments with spaces that follow the leading tabs — that deliberate tab-then-space mix is conformant, not a violation. The canonical indentation unit per language: Swift/Kotlin/Java/TypeScript 4 spaces unless repo config says otherwise, Python 4 spaces (never tabs), Go tabs via `gofmt`, Rust 4 spaces via `rustfmt`.
- **SHOULD NOT** commit trailing whitespace on any line.
- **SHOULD** end every text file with exactly one trailing newline — no missing final newline, no multiple blank lines at EOF.
- **SHOULD NOT** contain consecutive blank lines beyond the language convention (≤1 inside a function per F5; ≤2 at top level); three or more SHOULD NOT appear anywhere.
- **SHOULD NOT** use alignment padding — multiple spaces inserted to vertically align `=`, `:`, or trailing comments across lines. *Escape hatch: a genuine fixed-column data table (ASCII lookup, opcode map) where alignment IS the readability MAY align, isolated from logic.*

```python
# WRONG — column alignment; renaming one key reflows every line in the diff
config = {
    "host"        : "localhost",
    "port"        : 8080,
    "max_retries" : 3,
}

# RIGHT — single space after colon; a rename touches one line
config = {
    "host": "localhost",
    "port": 8080,
    "max_retries": 3,
}
```

## F9 — Line Length and Literal Readability

- **SHOULD NOT** exceed the project's per-language limit (F1 values); long lines SHOULD break at a natural boundary (after an operator, at a comma, before a `.` in a chain) — never mid-token. *Escape hatch: a single unbreakable token (a long URL in a comment, a base64 fixture, a generated string) MAY exceed the limit; it SHOULD sit on its own line so it never pushes logic off-screen.*
- **SHOULD** use the language's digit separator for long numeric literals (`1_000_000`).
- **SHOULD NOT** put magic numbers or magic strings inline in logic — extract a named constant (composes with Naming and Function-Shape). *Escape hatch: the universally-understood literals `0`, `1`, `-1`, `2`, and `""` MAY appear inline where self-evident (loop bounds, off-by-one, halving).*
- **SHOULD** break a long boolean or arithmetic expression before the operator (operator leads the continuation line) so the condition's structure is scannable.

```kotlin
// WRONG — unreadable literal and an over-wide condition
val limit = 10000000
if (user.isActive && user.hasPaid && !user.isSuspended && user.region == "EU" && user.tier > 2) {

// RIGHT — separated literal; condition wraps with the operator leading each continuation line
val limit = 10_000_000

if (user.isActive
    && user.hasPaid
    && !user.isSuspended
    && user.region == REGION_EU
    && user.tier > PREMIUM_TIER
) {
```

## Self-Review Checklist

| # | Check | Required | Rule |
|---|---|---|---|
| 1 | No signatures wrapped purely for visual symmetry | Required | F1 |
| 2 | All 4+ parameter, complex-type, or default-value signatures use multi-line format | Required | F1 |
| 3 | No ragged signature layout (whole single-line or whole one-param-per-line; tool-emitted signatures exempt) | Required | F1 |
| 4 | Call sites judged independently for single-line vs. multi-line | Required | F1 |
| 5 | Chains exceeding 2 calls broken across lines, one stage per line | Required | F2 |
| 6 | Chains of ≤2 calls stay on one line when they fit | Required | F2 |
| 7 | No partially-broken chains; reused intermediates hoisted to named bindings | Required | F2 |
| 8 | Multi-line single-expression functions use block body with explicit `return` | Required | F3 |
| 9 | Calls with 3+ args that exceed the line limit use multi-line argument layout | Required | F4 |
| 10 | No ragged multi-line argument lists (tool-emitted calls exempt) | Required | F4 |
| 11 | Unused parameters replaced with `_` | Required | F4 |
| 12 | Ambiguous positional args use labels/named args (escape hatch noted) | Required | F4 |
| 13 | Production-code body separated into Guard / Setup / Transform / Side-effect / Return, in order | Required | F5 |
| 14 | Exactly one blank line between blocks; no double blank lines inside a body | Required | F5 |
| 15 | No blank line inside a block, after `{`, or before `}` | Required | F5 |
| 16 | Out-of-order blocks reordered (or interleave justified by a comment) | Required | F5 |
| 17 | Test body separated into Arrange / Act / Assert (Given-When-Then) | Required | F5 |
| 18 | Exactly one Act block per test; no assertions before the Act | Required | F5 |
| 19 | Comments on line above, not end-of-line (sanctioned literal annotations exempt) | Required | F6 |
| 20 | Section markers use `// MARK: -` (Swift/Obj-C) or `// --- … ---` (other languages) | Required | F6 |
| 21 | No `// region` / `// endregion` / fold markers | Required | F6 |
| 22 | No commented-out code committed | Required | F6 |
| 23 | No comments that merely restate the code | Required | F6 |
| 24 | No reformat-only churn outside touched hunks | Required | F7 |
| 25 | No mixed reformatting + behavioral changes in one change set | Required | F7 |
| 26 | No whitespace-only churn riding a behavioral change | Required | F7 |
| 27 | No import/dependency-reorder churn riding an unrelated change | Required | F7 |
| 28 | Generated files committed as a separate labeled hunk, not hand-edited | Required | F7 |
| 29 | Consistent indentation unit; no tab/space mixing within a file | Required | F8 |
| 30 | No trailing whitespace; exactly one trailing newline at EOF | Required | F8 |
| 31 | No 3+ consecutive blank lines anywhere | Required | F8 |
| 32 | No alignment padding (fixed-column data tables exempt) | Required | F8 |
| 33 | No line exceeds the per-language limit (unbreakable-token escape hatch on own line) | Required | F9 |
| 34 | Long numeric literals use digit separators | Required | F9 |
| 35 | No inline magic numbers/strings (0, 1, -1, 2, "" exempt) | Required | F9 |
| 36 | Wrapped boolean/arithmetic expressions break before the operator | Required | F9 |
