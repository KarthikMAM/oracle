# Type Design

Well-designed types make illegal states unrepresentable, so the compiler — not a code reviewer, not a runtime assertion, not a customer — catches the defect. Poor types push validation into every consumer, where one forgotten check becomes an outage. **Posture (per `context/principles.md` → Simplicity):** prefer small types composed over deep inheritance, narrow interfaces over wide ones ("the bigger the interface, the weaker the abstraction"), and concrete return types over speculative abstraction — add an abstraction only when a real second case earns it. This is the Go ethos applied in each language's own idiom. Violations are BLOCK-severity for the clean-code reviewer lens.

## Anchors

- **SRP**, **OCP**, **ISP**, **DIP** (SOLID, Robert C. Martin) — one reason to change; open to extension, closed to modification; many small interfaces; depend on abstractions
- **LSP** (Liskov, 1987) — a subtype must be substitutable for its base without surprising the caller
- **Information Hiding** (Parnas, 1972) — a module hides a decision likely to change behind a stable interface
- **Make Illegal States Unrepresentable** (Scott Wlaschin) — encode invariants in the type so the wrong value cannot be constructed
- **Parse, Don't Validate** (Alexis King, 2019) — a constructor that rejects bad input returns a type that proves the input was good
- **Composition Over Inheritance** (Gamma et al.) — assemble behavior from parts; reserve inheritance for true substitutable subtyping
- **LCOM** (Chidamber & Kemerer, 1994) — low cohesion between a class's methods and its fields signals it is really two classes
- **Primitive Obsession** (Fowler, *Refactoring*) — a bare `String`/`int` carrying a domain concept is a defect waiting for a mismatched argument
- **Equality Contract** (Bloch — *Effective Java* items 10–11) — `equals` and `hashCode` are a joint contract; break it and hash-based collections silently lose data
- **CAP of Aliasing** (Hoare; Java's defensive-copy lore) — a mutable reference handed across a boundary is a shared-mutable-state bug the consumer cannot see

## Q4.1 — General

- **MUST** annotate every public/exported surface (exported function signatures, public properties, public return types, public stored fields) with explicit types. *Inference MAY be used for locals and `private` helpers where the initializer is on the same screen as its use.*
- **MUST** give each type exactly one responsibility; if it cannot be described in one sentence without "and," it MUST be split.
- **MUST** compose behavior via protocols/interfaces plus concrete implementations and inject dependencies through constructors; reaching for global/static mutable state or service locators inside a type MUST NOT occur.
- **MUST NOT** create new singletons outside the dependency registry, and a type MUST NOT cache an injected dependency in a `static`/`object`/module-level slot.
- **SHOULD** be immutable by default (`let` / `val` / `final` / `const`; `readonly` on TypeScript properties; `data class` with `val`); a mutable field MUST be documented at the declaration. *Escape hatch: performance-critical buffers, accumulators, and builder internals MAY be mutable when profiled or structurally required; confine the mutation to the smallest scope and document why.*
- **MUST** start visibility at the most restrictive level the language offers (`private`, then `internal`/module-private), promoting only for a real external consumer; a type or member promoted past `private` for testing only MUST NOT occur.
- **MUST NOT** represent a domain concept with a bare primitive when a mismatched value of the same primitive type would be a defect (order id vs user id; cents vs dollars; ms vs seconds; meters vs feet) — wrap it in a distinct type (newtype / branded type / value class). *Escape hatch: a primitive with no cross-type confusion risk and no invariant (a free-text `note`, a loop counter) MAY stay primitive.*

```kotlin
// WRONG — two responsibilities, bare-primitive ids, hidden global dependency
class UserReportService {                       // "manages users AND renders reports"
    fun email(userId: String, orderId: String) {
        val u = Database.global.find(userId)    // global mutable state, untestable
        Mailer.INSTANCE.send(u.email, render(orderId))
    }
}

// RIGHT — one responsibility, distinct id types, injected dependencies
@JvmInline value class UserId(val raw: String)
@JvmInline value class OrderId(val raw: String)

class OrderReceiptMailer(
    private val users: UserRepository,
    private val mailer: Mailer,
) {
    fun send(userId: UserId, orderId: OrderId) {
        val user = users.find(userId)
        mailer.send(user.email, render(orderId))   // orderId/userId cannot be swapped: compiler rejects it
    }
}
```

## Q4.2 — Type Centralization

- **SHOULD** extract and centralize types; a type literal with structure (an object/record shape, a multi-member union, a non-trivial function signature) SHOULD NOT appear inline in a parameter, return, or variable position where naming it would aid a reader or a second use. In languages where centralizing types is idiomatic (TypeScript/JS, Kotlin, Java), each module SHOULD have one types file (or `types/` directory) and cross-module types SHOULD live in a shared types package. *Go carve-out: Go defines types next to their use and interfaces at the consumer (Q4.14) — a single catch-all types file is an anti-pattern there; name a structural type, but keep it adjacent to its use.* *Do not extract a single-use alias that adds only indirection (principles.md → Simplicity: "clear over DRY", don't abstract until the occurrences earn it); name a shape when it repeats or when the inline form obscures the signature.*

Extract on sight (these forms SHOULD NOT appear inline):
- Object/record literal types (`{ key: Type }`, anonymous structs) anywhere
- Union types with 3+ members, or any union (even 2-member) used in 2+ locations
- Intersection types outside the types file
- Raw function/closure/lambda signatures in parameter or property positions
- Tuple types in return or parameter positions
- `Pair` / `Triple` / anonymous tuples as return types — use a named data class/record/struct
- Generic types with inline constraints — extract to a named interface
- Mapped/conditional/template-literal types (TypeScript) used in more than one place
- Inline enum-like string sets (`"a" | "b" | "c"`) referenced in 2+ locations
- A nested/anonymous struct with 2+ fields in a Go signature, or a multi-field anonymous closure-capture struct — name it (composes with Q4.14)

*Escape hatch: a `private` type used exclusively within a single class/file body MAY be a nested type. A single-use, single-location anonymous shape in a `private` local (e.g. a one-shot reducer accumulator never named in any signature) MAY stay inline; extract it before its second use or its first appearance in any non-private signature — before the drift, not after.*

```typescript
// WRONG — inline object type, inline closure type, inline union duplicated across signatures
function update(u: { id: string; role: "admin" | "user" }, onDone: (ok: boolean) => void) {}
function remove(u: { id: string; role: "admin" | "user" }, onDone: (ok: boolean) => void) {}
function audit(role: "admin" | "user") {}        // union + callback both duplicated; add a value and these drift

// RIGHT — extracted, named, single source of truth (each name earns its keep by 2+ uses)
// types/user.ts
export type Role = "admin" | "user"
export interface UserPatch { id: UserId; role: Role }
export type DoneCallback = (ok: boolean) => void

// user-service.ts
function update(patch: UserPatch, onDone: DoneCallback): void {}
function remove(patch: UserPatch, onDone: DoneCallback): void {}
function audit(role: Role): void {}
```

## Q4.3 — TypeScript-Specific

- **MUST** enable `"strict": true`, plus `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, and `noImplicitOverride`. *Escape hatch: a legacy package mid-migration MAY stage these on per-file via a tracked checklist, never by relaxing the project default.*
- **MUST NOT** use `any` (explicit or implicit), including `as any`, `any[]`/`Array<any>`, `Promise<any>`, `Record<string, any>`, and `any` type arguments — use `unknown` and narrow with a type guard.
- **MUST NOT** use the non-null assertion (`!`) or definite-assignment assertion (`!:`) — handle nullability with a guard or a parse step that proves presence.
- **MUST NOT** use type assertions (`as`) to widen, force, or reshape a value, and `as unknown as T` double-casts MUST NOT appear; the only sanctioned assertions are `as const` and narrowing to a *more specific* type immediately after a runtime type guard. Use `satisfies` for conformance without widening. *Escape hatch: an `as` at a genuine type-system boundary (a validated parse result, a generated-code seam, a DOM/`EventTarget` cast guarded by an `instanceof` check) MAY be used when accompanied by the runtime check that makes it sound, carrying the Q4.10 4-field exception block (which rule, why, scope, mitigation — the mitigation being that runtime check) adjacent to the cast.*
- **MUST NOT** use compiler-silencing directives (`@ts-ignore`, `@ts-expect-error`, `@ts-nocheck`); fix the type. *Escape hatch: `@ts-expect-error` MAY annotate a deliberately-invalid line inside a type-level test, and MUST carry a trailing description of the expected error (a bare `@ts-expect-error` MUST NOT be used).*
- **MUST** model multi-shape values as a discriminated union with a literal discriminant, narrowed by `switch` on the discriminant; the `switch` over an in-module discriminated union MUST be exhaustive, enforced by a `never`-typed default arm.
- **SHOULD** prefer `as const` objects or string-literal unions over `enum`; `const enum` MUST NOT be used. *Escape hatch: a numeric `enum` MAY be used only where an external wire/protocol contract pins the numeric values; pin each value explicitly and never rely on auto-increment ordinals (composes with Q4.14).*
- **MUST NOT** use `Object`, `Function`, `{}`, or `object` as types — use a precise interface, `Record<K, V>`, or `unknown`; a bare `Promise` without a type argument and `JSON.parse`'s implicit `any` result MUST be assigned to `unknown` and narrowed.
- **MUST NOT** mix `null` and `undefined` for the same concept — one null convention per project (`undefined` preferred).
- **MUST NOT** use an index signature (`{ [k: string]: T }`) where the keys are statically known — use an explicit interface or `Record<LiteralUnion, T>`.
- **SHOULD** declare a function-typed property as a `readonly` property arrow (`onX: (arg: T) => void`) rather than the bivariant method shorthand (`onX(arg: T): void`).

```typescript
// WRONG — any, non-null assertion, force-cast, illegal-state booleans
function parse(input: any): User {
  const raw = JSON.parse(input) as User            // force-cast: no runtime check at all
  return { ...raw, id: raw.id! }                    // non-null assertion papers over missing field
}
interface Req { loading: boolean; data?: User; error?: Error } // loading+data+error all settable at once

// RIGHT — unknown + guard, discriminated union makes illegal states unrepresentable
function parse(input: unknown): User {
  if (!isUser(input)) throw new ParseError("user")  // runtime guard proves the shape
  return input                                      // narrowed by isUser; no assertion needed
}
type RequestState =
  | { readonly status: "loading" }
  | { readonly status: "success"; readonly data: User }
  | { readonly status: "error"; readonly error: Error } // only one shape is ever representable

function render(state: RequestState): View {
  switch (state.status) {
    case "loading": return spinner()
    case "success": return view(state.data)
    case "error":   return errorView(state.error)
    default: { const _exhaustive: never = state; return _exhaustive } // new variant => compile error
  }
}
```

> **Class design** — the *class* as a unit of responsibility and collaboration (declaration order, sizing ceilings, cohesion, composition-over-inheritance, LSP, Tell-Don't-Ask, the Law of Demeter, constructor injection, and the God Object / Feature Envy / Anemic Model anti-patterns) is owned by **class-design.md (C1–C6)**, loaded whenever a change defines or changes a class. This file owns type-*level* design; the two compose but do not restate each other.

## Q4.5 — Swift-Specific

- **Value semantics by default** — **MUST** prefer `struct` over `class` unless reference identity or inheritance is genuinely required, and `let` over `var`; a non-`open` public-API `class` MUST be marked `final`. *Escape hatch: identity-bearing types (view controllers, types holding OS handles, deliberately shared coordinators) are legitimate classes — state the identity reason.*
- **Sendable at concurrency boundaries** — **MUST** conform any type crossing an actor or task boundary to `Sendable` (or be an `actor`); `@unchecked Sendable` MUST NOT be used without the Q4.10 4-field exception block adjacent to the declaration (the mitigation being the manual synchronization that makes it sound).
- **Exhaustive enums** — **MUST** make a `switch` over an in-module domain `enum` exhaustive without a catch-all `default` (a `@frozen`/library-evolution boundary is the exception, requiring a `default`); an associated-value `enum` SHOULD be preferred over a struct-of-optionals when cases are mutually exclusive (composes with Q4.8).
- **No force unwrap in type design** — **MUST NOT** use implicitly-unwrapped optionals (`T!`) on stored properties to defer initialization; use a real optional or a proper initializer (composes with Function-Shape Q3.1). *Escape hatch: `@IBOutlet` and framework-mandated IUO properties only.*
- **Protocol witness over inheritance** — **MUST** model capabilities as protocols with `struct` conformers rather than class hierarchies; a protocol with associated-type or `Self` requirements MUST NOT be used merely to type-erase a heterogeneous collection — use a concrete enum or an explicit type-eraser, not `Any`.
- **Equatable/Hashable parity** — **MUST** derive or implement `Hashable` consistently with `Equatable` for a value type used as a `Set` member or `Dictionary` key (composes with Q4.11).
- **Access control** — **MUST** default to `internal`; `public`/`open` is a one-way ABI door — promote only with a real cross-module consumer.

```swift
// WRONG — reference type with no identity reason; IUO defers a guaranteed crash
class GridCell { var row: Int!; var col: Int! }  // aliasing bugs + nil-deref waiting to happen

// RIGHT — value type, fully initialized, Sendable + Hashable for free.
// Integer fields are safe in equality/hashing; a Double-bearing value type is NOT safe as a
// Set member / Dictionary key (NaN != NaN) — exclude/quantize the float or document it per Q4.11.
struct GridCell: Sendable, Hashable { let row: Int; let col: Int }
```

## Q4.6 — Kotlin-Specific

- **Sealed class/interface for finite-variant domains** — **MUST** model a closed set of variants as a `sealed` hierarchy (or `enum`); `when` over a `sealed`/`enum` subject MUST omit `else ->`.
- **Data-class immutability** — **MUST** make all `data class` properties `val`; a mutable collection (`MutableList`, `MutableMap`, array) MUST NOT be a constructor property — expose the read-only interface (`List`, `Map`) and copy defensively on the way in (composes with Q4.13).
- **No `lateinit` for invariant deferral** — **MUST NOT** use `lateinit var` to defer a value that is part of the type's invariant; use a constructor parameter, a nullable backing field with a checked accessor, or distinct lifecycle types (Q4.8). *Escape hatch: framework-injected fields (DI, test fixtures) where the framework guarantees assignment before first use MAY use `lateinit`; carry the Q4.10 4-field exception block adjacent to the declaration (the mitigation being the framework contract that guarantees assignment).*
- **equals/hashCode contract** — **MUST NOT** override `equals` without `hashCode` (and vice versa); a type with custom equality MUST be `final` — prefer `data class`, which MUST NOT be a hash-map key if any property is mutable through aliasing (composes with Q4.11).
- **Value classes for primitive obsession** — **SHOULD** wrap domain identifiers and units in `@JvmInline value class`, not bare `String`/`Long` (composes with Q4.1).
- **No platform-type leakage** — **MUST** assert a Java-interop platform type (`T!`) to `T` or `T?` at the boundary, never propagate it.
- **`Pair`/`Triple` as return types** — **SHOULD NOT** be used; prefer a named `data class` (composes with Q4.2).

```kotlin
// WRONG — mutable list leaks into a "value" type; nullability lost at the Java boundary
data class Cart(val items: MutableList<Item>)        // caller mutates the cart's internals freely
fun load(): Cart = Cart(legacyApi.fetchItems())      // fetchItems() returns List<Item>! — null-unsafe

// RIGHT — immutable view, defensive copy, nullability pinned at the boundary
data class Cart(val items: List<Item>) {
    constructor(source: List<Item>) : this(source.toList())   // defensive copy
}
fun load(): Cart {
    val items: List<Item> = legacyApi.fetchItems() ?: emptyList()   // platform type resolved here
    return Cart(items)
}
```

## Q4.7 — Java-Specific

- **Records as the default DTO/value type** (JEP 395) — **MUST** make immutable carriers `record`s; a `record` component MUST NOT be a mutable array or mutable collection without a defensive copy in a compact constructor (and a copy on the accessor) (composes with Q4.13).
- **Sealed interfaces for closed hierarchies** (JEP 409) — **MUST** make closed type families `sealed`, and a `switch` over them SHOULD use exhaustive pattern matching with no `default`.
- **Optional discipline** — **MUST** use `Optional<T>` in return positions ONLY (never as a field, constructor parameter, method parameter, or collection element); `Optional.get()` MUST NOT be called without an `isPresent()`/`isEmpty()` guard, `==` MUST NOT compare `Optional`s, and `Optional` MUST NOT itself be null.
- **`equals`/`hashCode` together** — **MUST NOT** override one without the other; prefer a `record`, which generates both correctly (composes with Q4.11).
- **No raw types** — **MUST** parameterize a generic type (`List<String>`, never raw `List`).
- **No `Cloneable`/`clone()` for value copying** — **MUST NOT** use `Cloneable` or `Object.clone()` to copy a value type; use a copy constructor, a static factory, or a `record`.
- **`Serializable` discipline** — **MUST NOT** implement `Serializable` speculatively; implement it only with a real serialization requirement, a fixed `serialVersionUID`, and `readObject` validation that re-establishes invariants.
- **Null-intent annotations at the boundary** — **SHOULD** annotate nullable public API parameters and returns (`@Nullable`/`@NonNull`).

```java
// WRONG — Optional field, raw type, array leaks out of a "value" record
record Order(Optional<Coupon> coupon, List items, String[] tags) {} // 3 separate defects

// RIGHT — Optional only at return sites; parameterized; defensive copies guard immutability
record Order(Coupon coupon, List<Item> items, String[] tags) {
    Order {                                              // compact constructor: copy in
        items = List.copyOf(items);
        tags  = tags.clone();
    }
    public String[] tags() { return tags.clone(); }     // copy out: a component-accessor override MUST be public
    Optional<Coupon> couponIfPresent() {                // Optional appears only here, at a return
        return Optional.ofNullable(coupon);
    }
}
```

## Q4.8 — Make Illegal States Unrepresentable

- **MUST** model a set of mutually-exclusive states as a sum type (discriminated union / sealed hierarchy / enum-with-payload), not as a product of independent optional or boolean fields.
- **MUST** validate a value with a construction invariant (non-empty, in-range, well-formed) once in a constructor/smart-constructor that rejects invalid input, returning a type that proves the invariant thereafter (Parse, Don't Validate); re-validating at each use MUST NOT be the design.
- **MUST** model "valid only after a lifecycle step" with distinct types per phase (`UnvalidatedOrder` → `ValidatedOrder`) rather than nullable fields plus an `isValidated` flag.
- **MUST** make the enforcing constructor/smart-constructor the *only* construction path — public field assignment, a public no-arg constructor, or a mutable setter that can re-break the invariant MUST NOT coexist with it.
- **MUST NOT** model a "this or that, never both, never neither" relationship as two optionals; model it as a sum type.
- *Escape hatch: an invariant that genuinely cannot be expressed in the host type system (a cross-field arithmetic relation the language cannot encode) MAY be enforced by a constructor assertion; the constructor still MUST reject invalid input, and the un-encodable part MUST be documented at the type with a test pinning it.*

```typescript
// WRONG — product of optionals admits impossible combinations
interface Payment {
  cardToken?: string      // card path
  bankIban?: string       // bank path
  isPaid?: boolean        // paid with... which? both set? neither?
}

// RIGHT — sum type: every representable value is a valid state
type Payment =
  | { readonly method: "card"; readonly token: CardToken }
  | { readonly method: "bank"; readonly iban: Iban }

// RIGHT — phase types: a Shipped order cannot be missing a tracking number
type DraftOrder    = { readonly status: "draft"; readonly items: readonly LineItem[] }
type ShippedOrder  = { readonly status: "shipped"; readonly items: readonly LineItem[]; readonly tracking: TrackingId }
function refund(order: ShippedOrder): Refund { /* tracking is guaranteed present by the type */ }
```

## Q4.9 — Interface Segregation, Generics, and Variance

- **MUST** expose only the methods a given consumer actually uses (ISP); a consumer that needs one method MUST NOT depend on a ten-method interface — segregate into role interfaces. *Escape hatch: a cohesive interface whose methods are always used together (an iterator's `hasNext`/`next`) is not a violation.*
- **SHOULD** carry the *weakest* bound the implementation requires on a generic parameter (`Iterable<T>` over `ArrayList<T>` in a parameter); return positions SHOULD be as *specific* as the implementation guarantees (composes with DIP).
- **MUST** state variance correctly where the language expresses it: producers covariant (Kotlin `out`, Java `? extends`, Swift covariant positions), consumers contravariant (Kotlin `in`, Java `? super`); a mismatched collection MUST NOT be forced through with a cast.
- **MUST NOT** declare a mutable generic container covariant (arrays-as-covariant in Java is the classic `ArrayStoreException` trap); a covariant view MUST be read-only.
- **MUST NOT** use unbounded wildcard/`any`-shaped generics to dodge a bound; supply the real bound.

```kotlin
// WRONG — fat interface forces a read-only consumer to depend on writes it never calls
interface UserStore {
    fun find(id: UserId): User
    fun save(user: User)
    fun deleteAll()                 // a read path must not depend on this
}

// RIGHT — segregated role interfaces; each consumer depends only on what it uses (ISP)
interface UserReader { fun find(id: UserId): User }
interface UserWriter { fun save(user: User) }

// RIGHT — correct variance. `List<out E>` is already covariant at its declaration site, so a
// producer of any subtype is accepted without a use-site projection (a redundant `out` here would
// draw a compiler warning — see Function-Shape Q3.1 on compiler-silencing):
fun loadAll(readers: List<UserReader>): List<User> = readers.map { it.find(currentId) }

// A use-site `out` is load-bearing only on an *invariant* container (`MutableList<E>` is invariant):
fun loadAll(readers: MutableList<out UserReader>): List<User> = readers.map { it.find(currentId) }
```

## Q4.10 — Documented Exception Block

- **MUST** carry an inline comment adjacent to the declaration for any type or member that deviates from a SHOULD rule (mutable field, direct primitive for a domain concept, `@unchecked Sendable`, a sanctioned `as`/`@ts-expect-error`, a framework `lateinit`, a numeric `enum` pinned to a wire contract), containing all of (this same four-field block is the format class-design.md C2 uses for an oversized-class deviation):

1. **Which rule** is being excepted (the Q4.x id).
2. **Why** the conforming form does not work here.
3. **Scope** — exactly what the exception covers (this field, this class, this cast).
4. **Mitigation** — the compensating control (the runtime guard backing an `as`, the synchronization backing `@unchecked Sendable`, the framework contract backing a `lateinit`, the test pinning the invariant).

- **MUST** treat a deviation without all four fields as a violation, and the comment MUST sit adjacent to the declaration (a justification only in a commit message or CR thread MUST NOT count). MUST/MUST NOT safety rules (no `any`, LSP, inheritance depth, exhaustive sum types, defensive copies, equals/hashCode parity, single construction path) have no exception block.

```kotlin
// RIGHT — a mutable-field deviation made visible and approvable on sight
//
// TYPE-DESIGN EXCEPTION — Q4.1 (immutable by default)
// Why:        This accumulator is written once per row in a hot parse loop; a fresh
//             immutable copy per row was profiled as the dominant allocation cost.
// Scope:      This single `private var` running total, confined to parse().
// Mitigation: Mutation never escapes the method; the returned Total is immutable.
private var runningCents: Long = 0
```

## Q4.11 — Equality, Identity, and Hashing

- **MUST** override hashing consistently with equality and vice versa: `equals`+`hashCode` (Java/Kotlin), `==`+`hash(into:)` (Swift `Hashable`), `__eq__`+`__hash__` (Python), `PartialEq`/`Eq`+`Hash` (Rust); two values that compare equal MUST produce the same hash.
- **MUST** make a type used as a hash-set member or hash-map key immutable in all of its equality-relevant fields, or MUST NOT use it as a key.
- **MUST** make equality on a value type structural (by field value); identity equality (reference `===`, `ObjectIdentifier`, default `Object.equals`) MUST NOT be relied on for a value type.
- **MUST NOT** let a floating-point field silently participate in equality/hashing (where `NaN != NaN` and `+0.0`/`-0.0` hash differently); exclude it, quantize it, or document the semantics. For money and other exact quantities, an integer minor-unit or decimal type MUST be used instead of `float`/`double` (composes with Q4.1 units).
- **MUST** make a custom `equals` reflexive, symmetric, transitive, and consistent; an `equals` that compares across an inheritance boundary asymmetrically MUST NOT exist — prefer `final`/`data`/`record` value types.

```kotlin
// WRONG — equals overridden, hashCode not; the object vanishes from a HashSet
class Money(val cents: Long, val currency: String) {
    override fun equals(other: Any?) =
        other is Money && cents == other.cents && currency == other.currency
    // no hashCode(): two "equal" Money values land in different buckets -> set contains duplicates
}

// RIGHT — data class generates a consistent equals/hashCode pair over immutable vals
data class Money(val minorUnits: Long, val currency: CurrencyCode)   // integer money, not Double
```

## Q4.12 — Nullability and Optionality as Type Design

- **MUST** make an optional field optional in the type system (`T?`, `T | undefined`, `Optional<T>` in returns, `Option<T>`), never a sentinel value of the same type (`-1`, `""`, `0`, `Long.MIN_VALUE`).
- **MUST NOT** use `null`/`nil`/`undefined` to encode more than one distinct meaning; "not loaded yet" vs "loaded and empty" vs "load failed" are three representable values (a sum type, Q4.8).
- **MUST NOT** make optionality deep: a `T??`, a `List<T?>` where the inner null is also meaningful, or an `Optional<Optional<T>>` MUST be flattened into one explicit type with named cases.
- **SHOULD** minimize optional fields on a public type — many independent optionals is usually a union of cases in disguise (composes with Q4.8). *Escape hatch: a genuine sparse-attributes carrier (a patch/PATCH payload where any subset of fields may be present) legitimately has many optionals; model it as a dedicated `Patch` type distinct from the full entity, and document the sparse semantics.*
- **MUST NOT** silently coalesce a nullable field to a default at the point of *reading* when the default changes behavior — coalesce once at the parse boundary (Q4.8).

```typescript
// WRONG — sentinel + overloaded null; "-1" is a real index somewhere, null means two things
interface Session {
  expiresAtMs: number        // -1 == "never"? 0 == "expired"? a real timestamp?
  user: User | null          // not-logged-in? or logged-in-but-not-loaded? consumers guess
}

// RIGHT — optionality in the type, distinct states modeled, no sentinels (composes with Q4.8)
type Expiry = { readonly kind: "never" } | { readonly kind: "at"; readonly atMs: number }
type SessionUser =
  | { readonly kind: "anonymous" }
  | { readonly kind: "loading" }
  | { readonly kind: "authenticated"; readonly user: User }
interface Session { readonly expiry: Expiry; readonly user: SessionUser }
```

## Q4.13 — Collections and Mutable References in Type Signatures

- **MUST** expose the read-only collection interface on a public field, parameter, or return type, not a mutable one: `List`/`Map`/`Set` not `MutableList`/`MutableMap` (Kotlin), `List`/`Map` not `ArrayList`/`HashMap` (Java), `readonly T[]`/`ReadonlyArray<T>`/`ReadonlyMap` not `T[]`/`Map` (TypeScript), `&[T]`/`impl Iterator` over an owned `Vec` where ownership is not transferred (Rust).
- **MUST** copy a caller-supplied collection defensively on the way in, and an accessor returning an internally-mutable collection MUST copy it on the way out (or return an unmodifiable view); an array component is the highest-risk case — clone it explicitly (composes with Q4.7).
- **MUST NOT** mutate a collection passed in as a parameter as a side effect (an "output parameter" collection); return a new collection instead (composes with Function-Shape Q3.4).
- **SHOULD** prefer the most precise read-only collection that states intent: a non-empty list invariant MUST be a `NonEmptyList` type, not a `List` plus a runtime check (composes with Q4.8). *Escape hatch: a hot path that has profiled the defensive copy as a real cost MAY share an immutable, never-mutated collection by reference; document the immutability guarantee at the boundary.*
- **MUST NOT** put a `Map`/`Set` whose key type has unstable equality (Q4.11) in a public signature.

```java
// WRONG — constructor aliases the caller's list; the caller can mutate Roster's internals later
final class Roster {
    private final List<Player> players;
    Roster(List<Player> players) { this.players = players; }   // no copy: shared mutable state
    List<Player> players() { return players; }                 // leaks the live list out, too
}

// RIGHT — copy in, unmodifiable out; no live reference to internals ever escapes
final class Roster {
    private final List<Player> players;
    Roster(List<Player> players) { this.players = List.copyOf(players); }   // defensive copy in
    List<Player> players() { return players; }                              // List.copyOf is already unmodifiable
}
```

## Q4.14 — Enum, Constant, and Go/Rust Type Design

- **MUST** make a closed set of named choices an enum/sum type, not a set of `String`/`int` constants compared with `==` (composes with Q4.8); magic literals carrying a domain meaning SHOULD be named once rather than repeated inline (composes with Function-Shape Q3.3 constants).
- **MUST NOT** rely on ordinal/declaration order for identity in an enum that is serialized, persisted, or sent on the wire — pin an explicit stable value (a string name or an explicitly assigned number); Java `enum.ordinal()` and Kotlin `enum.ordinal` MUST NOT be persisted.
- **MUST NOT** use a default/`else`/`_` arm in a `switch`/`when`/`match` over an in-module enum to "handle the rest" (composes with Q4.3, Q4.5, Q4.6, Q4.7); a genuinely open enum crossing a library-evolution boundary is the documented exception (Q4.5 `@frozen`).

**Go-specific:**
- **MUST** make a domain concept a defined type, not a bare alias that vanishes (`type UserID string`, then methods on it) (composes with Q4.1 / Q4.12 — Go has no `Option`, so a pointer `*T` or an explicit "ok" boolean encodes absence, never a zero-value sentinel).
- **MUST NOT** store an exported struct field slice or map without copying — Go has no defensive immutability, so a shared slice header is aliased mutable state (composes with Q4.13). Copy on the way in.
- **MUST** define an interface at the *consumer* and keep it small (often one method); a fat interface defined at the producer is an ISP violation (composes with Q4.9), and the empty `interface{}`/`any` MUST NOT be used to dodge a type.

**Rust-specific:**
- **MUST** enforce a construction invariant by a private field plus a smart constructor returning `Result`, never a `pub` field that lets callers bypass it (composes with Q4.8).
- **MUST** use a newtype (`struct UserId(String)`) for a domain identifier over a bare `String`/`u64`, deriving the traits it actually needs (`Clone`, `PartialEq`, `Eq`, `Hash`) consistently (composes with Q4.1 / Q4.11).
- **MUST** keep a trait used as a `dyn` object object-safe and exposed only with the methods its consumer needs (composes with Q4.9).

```rust
// WRONG — magic strings compared by ==, public field bypasses the invariant
pub struct Order { pub status: String }            // "shipped"? "Shipped"? "SHIPPD"? all compile
if order.status == "shppd" { /* typo never caught */ }

// RIGHT — enum makes the choice closed; field is private with a checked constructor
pub enum OrderStatus { Draft, Shipped { tracking: TrackingId }, Cancelled }
pub struct Order { status: OrderStatus }           // exhaustive `match`; no stringly-typed status
```

```go
// WRONG — bare string id confusable with any other string; exported slice aliased
type Order struct {
    UserID string        // swappable with OrderID, both `string`
    Items  []Item        // caller keeps a reference and mutates Order's internals
}

// RIGHT — defined id type, defensive copy, absence via pointer not zero-value
type UserID string
type Order struct {
    userID UserID
    items  []Item
}
func NewOrder(userID UserID, items []Item) *Order {
    copied := make([]Item, len(items))
    copy(copied, items)                  // no shared slice header escapes
    return &Order{userID: userID, items: copied}
}
```

## Self-Review Checklist

| # | Check | Required | Rule |
|---|---|---|---|
| 1 | Every public/exported surface (incl. public stored fields) has explicit type annotations | Required | Q4.1 |
| 2 | One responsibility per type (no "and" in its description) | Required | Q4.1 |
| 3 | Constructor injection only; no global/static mutable state, service locators, or back-door cached singletons inside types | Required | Q4.1 |
| 4 | Immutable by default; each mutable field documented (escape hatch recorded) | Required | Q4.1 |
| 5 | Visibility starts at the most restrictive level; promoted only for a real consumer, never for tests | Required | Q4.1 |
| 6 | Domain concepts (incl. units) use distinct types, not confusable bare primitives | Required | Q4.1 |
| 7 | No inline object/union/intersection/tuple/mapped types in signatures | Required | Q4.2 |
| 8 | No raw closure/lambda/function signatures in params/properties | Required | Q4.2 |
| 9 | No `Pair`/`Triple`/anonymous-tuple (or multi-field anonymous struct) return types | Required | Q4.2 |
| 10 | All type definitions centralized in the types file/package | Required | Q4.2 |
| 11 | TS `strict` + `noUncheckedIndexedAccess` + `exactOptionalPropertyTypes` + `noImplicitOverride` enabled | Required | Q4.3 |
| 12 | No `any` in TypeScript (incl. `as any`/`any[]`/`Promise<any>`/`Record<string, any>`) — `unknown` narrowed with a guard | Required | Q4.3 |
| 13 | No non-null/definite-assignment assertion (`!`) in TypeScript | Required | Q4.3 |
| 14 | No `as` widening/force-casts; `satisfies` for conformance; sanctioned `as` backed by a runtime guard | Required | Q4.3 |
| 15 | No `@ts-ignore`/`@ts-nocheck`; `@ts-expect-error` only in type-level tests with a description | Required | Q4.3 |
| 16 | Multi-shape values are discriminated unions with `never`-checked exhaustive `switch`; no `const enum`; no `{}`/`Object`/`object`/`Function`; bare `Promise`/`JSON.parse` results narrowed; one null convention; no static-key index signatures | Required | Q4.3 |
| 17 | Swift: struct over class unless identity required; public classes `final`; no IUO stored props; `Sendable` at boundaries; no `Any`-erased collections | Required | Q4.5 |
| 18 | Swift: domain enum `switch` exhaustive without `default` (in-module); Equatable/Hashable parity on keys | Required | Q4.5 |
| 19 | Kotlin: sealed/enum for finite variants; `when` omits `else` on sealed/enum; no invariant-deferring `lateinit` | Required | Q4.6 |
| 20 | Kotlin: data-class properties all `val`; no mutable-collection constructor properties | Required | Q4.6 |
| 21 | Kotlin: `equals` not overridden without `hashCode`; platform types resolved at the boundary | Required | Q4.6 |
| 22 | Java: records as default value type with defensive copies; no `Cloneable`/`clone()`; `Serializable` only with real need + `serialVersionUID` + `readObject` validation | Required | Q4.7 |
| 23 | Java: `Optional` only in return positions, guarded, never null; no raw generic types | Required | Q4.7 |
| 24 | Illegal states unrepresentable: sum types over flag/optional products; invariants parsed once; single construction path; no this-or-that-via-two-optionals | Required | Q4.8 |
| 25 | Lifecycle phases modeled as distinct types, not nullable-field + flag | Required | Q4.8 |
| 26 | Interfaces segregated (ISP); generics carry the weakest needed bound (specific returns); variance stated correctly; mutable containers not covariant | Required | Q4.9 |
| 27 | Every SHOULD deviation carries a 4-field exception block (rule, why, scope, mitigation) adjacent to the declaration | Required | Q4.10 |
| 28 | `equals`/`hashCode` (and language equivalents) overridden together and mutually consistent; equality is structural for value types | Required | Q4.11 |
| 29 | Hash keys/set members immutable in equality-relevant fields; no float in keys; money as integer/decimal; equals reflexive/symmetric/transitive | Required | Q4.11 |
| 30 | Optionality is in the type, not a sentinel; `null`/`nil` encodes exactly one meaning; no nested optionality; coalesce once at the boundary | Required | Q4.12 |
| 31 | Public signatures expose read-only collections; defensive copy in/out; no collection output-parameter mutation; non-empty modeled as a type | Required | Q4.13 |
| 32 | Enums/constants for closed choices; no persisted ordinals; Go defined-types + copied exported slices + consumer-side small interfaces; Rust newtypes + private-field smart constructors + object-safe traits | Required | Q4.14 |
