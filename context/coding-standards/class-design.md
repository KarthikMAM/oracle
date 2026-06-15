# Class Design

A class is a contract about *responsibility and collaboration*: what one concept owns, the invariants it guarantees, and how little of itself it exposes to do its job. A class that does too much, reaches across its collaborators, or lets a subclass break its parent's promise is a defect that compiles — the kind that surfaces as a tangled change months later when no single edit is safe. **Posture (per `context/principles.md` → Simplicity):** small types with one reason to change, composed rather than inherited; narrow interfaces over wide ones; behavior next to the data it guards; the simplest shape that holds the invariant. This is the Go ethos applied in each language's own idiom. Type-*level* concerns — illegal states, primitives, nullability, sum types, equality — are owned by `type-design.md`; this file owns the *class* as a unit of responsibility and collaboration. Violations are BLOCK-severity for the clean-code reviewer lens.

## Anchors

- **SRP** (Robert C. Martin) — one reason to change per class; if it needs "and" to describe, it is two classes
- **OCP / ISP / DIP** (SOLID) — open to extension, closed to modification; many small role interfaces; depend on abstractions, inject them
- **LSP** (Liskov, 1987) — a subtype must be substitutable for its base without surprising the caller
- **Composition Over Inheritance** (Gamma et al.) — assemble behavior from parts; reserve inheritance for true substitutable subtyping
- **Information Hiding** (Parnas, 1972) — a class hides a decision likely to change behind a stable interface
- **LCOM** (Chidamber & Kemerer, 1994) — low cohesion between a class's methods and its fields signals it is really two classes
- **Tell, Don't Ask** (Pragmatic Programmers) — tell an object to act on its own state; do not pull the state out to decide for it
- **Law of Demeter** (Lieberherr, 1987) — talk to your immediate collaborators, not their internals
- **Anemic Domain Model** (Fowler, 2003) — a data bag whose behavior lives in a separate `*Service` is a procedural design wearing object clothes

## C1 — Single Responsibility and Cohesion

- **MUST** give each class exactly one responsibility; if it cannot be described in one sentence without "and," it MUST be split.
- **SHOULD** keep cohesion high — every method touches more than half of the stored state, or the class SHOULD be split (see Feature Envy, C6). A cluster of methods that touch one subset of fields and another cluster that touch a different subset is two classes wearing one name.
- **MUST** keep data and the behavior that enforces its invariants in the same class — no anemic domain model where a `*Service` holds all the logic for a property-bag type (C6).
- **MUST** treat an *infrastructure mechanism* a class needs — a queue, a cache, a retry buffer, a connection pool, a rate limiter, a scheduler — as a **separate responsibility from the business logic**, owned by its own type behind a narrow interface and injected (C5). The domain class calls `queue.enqueue(job)`; it MUST NOT embed the mechanism's mechanics (the backing buffer, capacity/backpressure, the dequeue loop, retry/drain bookkeeping) in its own body. The business rule is "what to enqueue and when"; how the queue holds, bounds, and drains work is the queue's concern. *Escape hatch: a class whose one responsibility genuinely *is* the mechanism (a `BoundedWorkQueue`, an `LruCache`) owns those mechanics — that is its single responsibility, and it carries no business logic in return.*

```kotlin
// WRONG — two responsibilities ("manages users AND renders reports"); zero cohesion
class UserReportService {
    fun emailReceipt(userId: UserId, orderId: OrderId) { /* user lookup + mailing */ }
    fun renderMonthlyReport(month: Month): Report { /* unrelated reporting */ }
}

// RIGHT — one responsibility each; every method works the class's own state
class OrderReceiptMailer(private val users: UserRepository, private val mailer: Mailer) {
    fun send(userId: UserId, orderId: OrderId) {
        val user = users.find(userId)
        mailer.send(user.email, receiptFor(orderId))
    }

    private fun receiptFor(orderId: OrderId): Email = /* ... */
}
```

The subtler split is logic from *mechanism*: a class that needs a queue must not become a queue.

```kotlin
// WRONG — business logic muddled with the queue's mechanics; two responsibilities in one class
class OrderProcessor(private val payments: PaymentGateway) {
    private val buffer = ArrayDeque<Order>()      // the queue's storage...
    private val capacity = 1000                   // ...capacity...
    private var draining = false                  // ...and drain bookkeeping all leak in here

    fun submit(order: Order) {
        if (buffer.size >= capacity) buffer.removeFirst()   // backpressure policy buried in domain code
        buffer.addLast(order)
        if (!draining) drainLoop()                          // the processor now also IS the queue
    }
    private fun drainLoop() { /* dequeue + retry + mark draining... */ }
}

// RIGHT — the queue is its own responsibility behind a narrow interface, injected (C5).
// OrderProcessor holds only the business rule: which orders to enqueue.
interface WorkQueue<T> { fun enqueue(item: T) }      // mechanics (buffer/capacity/drain) live in the impl

class OrderProcessor(private val queue: WorkQueue<Order>) {
    fun submit(order: Order) {
        if (order.isPayable) queue.enqueue(order)    // pure business decision; no queue mechanics
    }
}
```

## C2 — Class Shape and Sizing

Class bodies **SHOULD** follow this canonical declaration order:

| # | Section | Contents |
|---|---|---|
| 1 | **Companion / Static** | Constants, factory methods, type aliases |
| 2 | **Stored State** | `val`/`let` properties (immutable first), then mutable |
| 3 | **Initializer** | Primary constructor, secondary constructors, `init` blocks |
| 4 | **Public API** | Methods this class exists to expose, ordered by importance |
| 5 | **Helper / Internal** | `private`/`internal` methods supporting the public API |
| 6 | **Nested types** | Companion types, sealed-class members, nested data classes |

### Sizing limits

- **SHOULD NOT** exceed **150 lines** of class length (excluding blank lines and doc comments).
- **SHOULD NOT** exceed **10** public methods.
- **SHOULD NOT** exceed **6** stored properties.
- **SHOULD NOT** exceed **4** constructor parameters (else use a builder / parameter object); a parameter object SHOULD be a named type (type-design.md Q4.2) and MUST NOT be a grab-bag that re-imports the God Object you just split out.
- **SHOULD NOT** exceed **20 lines** per public method (inherits Function-Shape Q3.1), and nesting/complexity inside methods inherits Function-Shape Q3.1–Q3.2 (≤1 nesting level, cyclomatic ≤5, cognitive ≤7).

*Escape hatch (sizing only): a class that legitimately exceeds a SHOULD ceiling — a state machine with one method per state, a generated client, the host of an exhaustive `sealed` hierarchy — MAY do so when (a) it still has exactly one responsibility and (b) it carries the 4-field exception block of type-design.md Q4.10 (which rule, why, scope, mitigation) adjacent to the class declaration.*

## C3 — Composition Over Inheritance, and LSP

- **MUST** compose behavior from injected parts rather than inherit it; reserve inheritance for a true substitutable *is-a* whose subtypes honor the base contract.
- **MUST NOT** exceed **1** level of inheritance depth from a non-abstract concrete parent — concrete-on-concrete past one level MUST be replaced with composition. *No escape hatch.*
- **MUST** honor LSP: replace with composition any subtype that throws where the base does not, returns `null`/`nil` where the base guarantees non-null, strengthens a precondition, or weakens a postcondition. A subtype that no-ops part of the parent's contract (Refused Bequest, C6) is not an *is-a*.
- **SHOULD** model a capability as an interface/protocol with concrete conformers, not a base class — many small role interfaces (ISP), each consumer depending only on the methods it calls.

```typescript
// WRONG — inheritance for code reuse; the subtype violates the base contract (LSP break)
class Rectangle { constructor(public width: number, public height: number) {}
  area(): number { return this.width * this.height } }
class Square extends Rectangle {                 // a Square is-NOT-substitutable-for a Rectangle
  set width(w: number) { super.width = super.height = w }   // mutating width also changes height: surprises callers
}

// RIGHT — compose the shared capability behind a narrow interface; no surprising subtype
interface Shape { area(): number }
class Rectangle implements Shape { constructor(private readonly w: number, private readonly h: number) {}
  area(): number { return this.w * this.h } }
class Square implements Shape { constructor(private readonly side: number) {}
  area(): number { return this.side * this.side } }
```

## C4 — Encapsulation: Tell, Don't Ask, and the Law of Demeter

- **MUST** move a decision inside the class when a caller would otherwise pull the class's state out to make it (Tell, Don't Ask); a public getter that exists only so an external function can recompute what the class already knows MUST NOT be added.
- **MUST NOT** reach across object boundaries with chains like `a.b.c.d()` — ask the immediate collaborator, which asks its own. *Escape hatch: fluent builders, query/DSL builders, and stdlib collection-pipeline chains (`list.filter { }.map { }.first()`) are not Demeter violations — they operate on one conceptual subject.*
- **MUST** start every member at the most restrictive visibility the language offers (`private`, then `internal`/module-private), promoting only for a real external consumer; a member promoted past `private` for testing only MUST NOT exist.
- **MUST NOT** expose a `protected`/`open` mutable field to subclasses — expose a `protected` accessor, or keep the state `private` behind a guarded mutator. *No escape hatch.*

```kotlin
// WRONG — Ask: the caller pulls state out and makes the class's decision for it; Demeter chain
fun describeDiscount(order: Order): String {
    if (order.customer.account.tier.isPremium) {       // a.b.c.d reach-through
        return "10% off"
    }
    return "no discount"
}

// RIGHT — Tell: the object answers about its own state; no reach-through
fun describeDiscount(order: Order): String = order.discountLabel()   // Order asks its own customer
```

## C5 — Construction and Dependency Injection

- **MUST** have dependencies arrive via the primary constructor; setter injection, field reflection, and service locators MUST NOT be used.
- **MUST NOT** create new singletons outside the dependency registry, and a class MUST NOT cache an injected dependency in a `static`/`object`/module-level slot.
- **MUST NOT** design a two-phase `init()` / `start()` that leaves the object in an invalid state between construction and the second call; a constructed object is a fully valid object. *Model "valid only after a lifecycle step" as distinct types per phase (type-design.md Q4.8), not a half-built object plus an `isReady` flag.*
- **SHOULD** keep the constructor free of work beyond storing dependencies and establishing invariants — no I/O, no global lookups, no heavy computation in a constructor.

```swift
// WRONG — two-phase init leaves an invalid object; dependency grabbed from a global
final class ReportBuilder {
    private var repo: OrderRepository!          // nil until start() — every method risks a crash
    func start() { repo = ServiceLocator.shared.orders }   // hidden global dependency
}

// RIGHT — dependencies injected; the object is valid the instant it exists
final class ReportBuilder {
    private let repo: OrderRepository
    init(repo: OrderRepository) { self.repo = repo }
}
```

## C6 — Anti-Patterns to Flag

Each of these MUST be refactored when found:

- **God Object** — exceeds the C2 sizing ceilings without a documented escape, or needs "and" to describe. Split by responsibility.
- **Feature Envy** — a method uses another type's data more than its own; move it to that type.
- **Anemic Domain Model** — a domain type that is a property bag while a `*Service` holds all its logic; move the logic onto the domain type (C1).
- **Data Class smell** — only data, no behavior; either grow the behavior that owns its invariants, or make it an explicit immutable value type (record / `data class`) whose role as a pure value is intentional.
- **Lazy Class** — too small to justify its existence and forwards everything; inline it into its sole caller (composes with Function-Shape Q3.5).
- **Middle Man** — a class that delegates >80% of its methods straight through to one collaborator with no added behavior; remove it and let callers talk to the collaborator (composes with Lazy Class).
- **Refused Bequest** — a subclass that ignores or no-ops part of its parent's contract; prefer composition (C3, LSP).
- **Boolean-blind / flag-driven object** — a class whose valid states are encoded as combinations of independent booleans (`isOpen`, `isClosed`, `isPending` that can all be true at once); replace with a sealed/enum state (type-design.md Q4.8).

```swift
// WRONG — anemic data + Feature-Envy service; illegal states via independent flags
struct Account { var balanceCents: Int; var isFrozen: Bool; var isClosed: Bool } // frozen AND closed?
enum AccountService {
    static func withdraw(_ a: inout Account, _ cents: Int) {                       // envies Account's data
        a.balanceCents -= cents                                                    // no invariant enforced
    }
}

// RIGHT — behavior lives with the data; state is a single enum; invariant enforced in one place.
// (Swift `throws` with a typed error caught at the edge is the sanctioned idiom here — error-handling.md;
// the lesson is encapsulation + the single status enum, not the error-signalling mechanism.)
struct Account {
    private(set) var balanceCents: Int
    private(set) var status: AccountStatus

    mutating func withdraw(_ cents: Int) throws {
        guard status == .open else { throw AccountError.notOpen }
        guard cents <= balanceCents else { throw AccountError.insufficientFunds }
        balanceCents -= cents
    }
}
enum AccountStatus { case open, frozen, closed }   // exactly one status; "frozen AND closed" is impossible
```

## Self-Review Checklist

| # | Check | Required | Rule |
|---|---|---|---|
| 1 | One responsibility per class (no "and" in its description) | Required | C1 |
| 2 | High cohesion — every method touches >50% of stored state, or the class is split | Required | C1 |
| 3 | Data and the behavior that enforces its invariants live in the same class (no anemic model) | Required | C1 |
| 4 | Infrastructure mechanism (queue/cache/pool/limiter) is a separate injected collaborator, not mechanics inlined into the domain class | Required | C1 |
| 5 | Class members ordered Companion / Stored / Init / Public / Private / Nested | Required | C2 |
| 6 | Class ≤150 lines, ≤10 public methods, ≤6 fields, ≤4 ctor params (or documented escape) | Required | C2 |
| 7 | Public methods ≤20 lines; method nesting/complexity inherits Function-Shape Q3.1–Q3.2 | Required | C2 |
| 8 | Composition preferred over inheritance; inheritance only for a true substitutable is-a | Required | C3 |
| 9 | Inheritance depth ≤1 from a concrete parent (no escape) | Required | C3 |
| 10 | LSP honored — no throw/null/precondition/postcondition surprises in subtypes | Required | C3 |
| 11 | Capabilities modeled as small role interfaces (ISP), each consumer depending only on what it uses | Required | C3 |
| 12 | Tell-Don't-Ask honored — no getter that exists only for an external function to recompute the class's own state | Required | C4 |
| 13 | No Law-of-Demeter reach-through chains (builder/query/pipeline chains exempt) | Required | C4 |
| 14 | Visibility starts at the most restrictive level; promoted only for a real consumer, never for tests | Required | C4 |
| 15 | No `protected`/`open` mutable state exposed to subclasses (no escape) | Required | C4 |
| 16 | Constructor injection only; no setter injection, reflection, or service locators | Required | C5 |
| 17 | No new singletons outside the dependency registry; no injected dependency cached in a static slot | Required | C5 |
| 18 | No two-phase init leaving an invalid state; a constructed object is fully valid (phases as distinct types) | Required | C5 |
| 19 | No God Object / Feature Envy / Anemic Domain / Data Class smell / Lazy Class / Middle Man / Refused Bequest / boolean-blind object | Required | C6 |
