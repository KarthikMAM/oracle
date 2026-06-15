# Naming

Names are the primary documentation: every reader pays the cost of a bad name on every read, and a misleading name causes the subtle defects that survive review because the code *looks* right. A symbol whose name lies about its role, units, or lifetime is a latent production incident. Violations are BLOCK-severity for the clean-code reviewer lens.

## Anchors

- **Tim Ottinger — *Naming Is Everything* (2008)** — names communicate role and intent, not type or mechanism
- **Kent Beck — *Implementation Patterns* / Code Smells (2007)** — obscure names and long methods are the two strongest decay signals
- **Joshua Bloch — *How to Design a Good API and Why It Matters* (2006)** — names should be self-explanatory; the doc comment exists for the contract, not to rescue a bad name
- **Phil Karlton (attrib.)** — the two hard problems are cache invalidation and naming things; treat naming as load-bearing, not cosmetic
- **Robert C. Martin — *Clean Code* (2008), ch. 2** — intention-revealing, pronounceable, searchable names; no disinformation
- **Linguistic Antipatterns (Arnaoudova et al., 2013)** — names that contradict behavior (a getter that mutates, an `is*` that returns non-boolean) measurably raise defect density

## Q3.3 — Naming

A name MUST NOT lie — the correctness spine. Every rule below is a specialization of it, and each carries its own escape hatch so it never fights a correct edge case.

### Lexical Conventions by Symbol Kind

- **SHOULD** · a reader infers a symbol's kind from its name (per the table) without jumping to the declaration.

| Symbol | Convention | Examples |
|---|---|---|
| Functions / methods | Verb phrase | `validate`, `fetchUser`, `parseResponse`, `dispatchEvent` |
| Pure query functions | Noun/adjective phrase or `get`-free verb that asserts no side effect | `activeUsers`, `isExpired`, `remainingBudget` |
| Classes / structs / types | Noun phrase | `BridgePool`, `TokenValidator`, `OrderReceipt` |
| Protocols / interfaces | Capability adjective or noun | `Cacheable`, `TokenProviding`, `Repository` |
| Booleans | `is` / `has` / `should` / `can` / `did` / `will` / `was` prefix, or an `all` / `any` / `every` quantifier for plural-subject predicates | `isValid`, `hasExpired`, `shouldRetry`, `canDelete`, `allValid` |
| Collections | Plural noun | `users`, `pendingRequests`, `activeConnections` |
| Maps / dictionaries | `<value>By<key>` or `<value>Per<key>` | `userById`, `priceByRegion`, `retriesPerHost` |
| Constants | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT`, `DEFAULT_TIMEOUT_MS` |
| Enum cases | Singular noun describing the state | `Pending`, `Shipped`, `Cancelled` |
| Type parameters / generics | Single capital or a descriptive noun when ≥2 | `T`, `Key`, `Value`, `Element` |
| Files | Match the primary export/type name exactly | `UserRepository.ts` exports `UserRepository` |

- **SHOULD** · loop indices `i`/`j`/`k`, the single-line lambda parameter (`it`, `_`, `x => x.id`), and well-established math symbols (`n`, `dx`) are exempt from descriptive-name conventions when their scope is ≤5 lines and entirely visible without scrolling — *a name that escapes that scope MUST be descriptive*.

```typescript
// WRONG — vague verb + pronoun-grade local
function check(thing: User): boolean
const tmp = users.filter(u => check(u))

// RIGHT — verb names the query, local named by what it holds
function isActive(user: User): boolean
const activeUsers = users.filter(isActive)
```

### Banned and Pronoun-Grade Names

- **SHOULD NOT** · use any of the following as the *primary* identifier of a type, module, function, field, or non-throwaway local.

| Banned | Why | Better |
|---|---|---|
| `Manager` | Doesn't describe responsibility | `OrderProcessor`, `SessionRegistry` |
| `Helper` / `Helpers` / `Util` / `Utils` | "Stuff that helps" — meaningless junk drawer | Named module by responsibility |
| `Service` (generic) | Says where it lives, not what it does | `EmailDispatcher`, `PaymentReconciler` |
| `Handler` (generic) | Same issue | `OrderEventHandler`, `RequestRouter` |
| `Processor` (generic) | Vague | `BatchOrderShipper`, `LogAggregator` |
| `Wrapper` | Says HOW, not WHAT | The thing it wraps, e.g., `RetryingHttpClient` |
| `Base` / `Abstract<X>` / `<X>Impl` | Inheritance / scaffolding smell | Extract to interface or compose; name by behavior |
| `Common` / `Shared` / `Core` (as a module) | Junk drawer | Specific module by concern |
| `Misc` / `Stuff` / `Things` | Junk drawer | Same |
| `Factory` (generic) | Says pattern, not product | `ConnectionPool`, or a plain `connect()` function |
| `do` / `perform` / `execute` / `run` / `handle` (alone) | Verb with no object | Name the action and its object |
| `Info` / `Data` / `Object` / `Entity` / `Item` / `Element` (as a suffix) | Says it is data — everything is | Name by what it models: `Receipt`, `AuditRecord` |
| `tmp` / `tmp1` / `data2` / `temp` / `foo` / `bar` / `test1` | Numbered / placeholder | Named by content |
| `result` / `value` / `val` / `item` / `obj` / `thing` / `arg` / `param` / `ret` | Pronoun-grade | Named by what it IS |
| `flag` / `mode` / `state` / `status` (bare) | Names the variable's role, not its meaning | `isEnabled`, `RetryPolicy`, `connectionState` |
| `list` / `arr` / `map` / `dict` / `str` / `num` (type-as-name) | Restates the type | Plural domain noun: `invoices`, `priceByRegion` |

- **SHOULD NOT** · *a banned word MAY appear as a qualified part of a compound name when it is the genuinely correct domain term and the qualifier carries the meaning (`WindowManager`, `StateMachine`, `ServiceAccount`); the bare word standing alone is never clearer. A foreign schema, wire field, or platform API name you cannot rename is matched at the boundary and isolated there.*

```typescript
// WRONG — junk-drawer module + pronoun-grade locals (utils.ts)
export function process(data: any): any {
  const result = data.map((x: any) => x.value)
  return result
}

// RIGHT — module and symbols named by responsibility (invoiceTotals.ts)
export function sumLineItems(lineItems: LineItem[]): Money {
  const amounts = lineItems.map(lineItem => lineItem.amount)
  return amounts.reduce((total, amount) => total.plus(amount), Money.zero("USD"))
}
```

### Names Must Not Lie About Side Effects, Cost, or Return

- **MUST NOT** · a `get*`, `is*`, `has*`, or noun-phrase accessor mutate observable state, perform I/O, block, or throw on the normal path — name the side effect instead (`loadUser`, `fetchOrThrow`, `computeAndCache`). *Escape hatch: an idiomatic lazy property getter that computes-and-caches on first read MAY keep its noun name when the memoization is observably pure (same value every read, no I/O, no externally visible mutation) and a one-line comment records that it is memoized; a platform/framework-mandated accessor name you cannot rename is matched at the boundary and not propagated inward.*
- **MUST** · a function named as a pure query return what the name promises — `is*`/`has*`/`should*`/`can*` return a boolean (or boolean-shaped optional), a plural-noun query returns a collection. *Escape hatch: a foreign API, generated DTO, or wire contract that fixes a misleading name (an `is*` field that is actually an enum) is matched at the boundary and immediately re-parsed into a truthfully named domain type; the foreign name MUST NOT travel inward.*
- **SHOULD NOT** · understate cost — an operation doing network or disk I/O, or O(n)+ over an unbounded collection, named like a cheap field access; prefer `fetch*`, `load*`, `query*`, `compute*` over `get*`. *Escape hatch: a platform convention where `get*` is idiomatic for a backed-but-cheap read (amortized cached property) MAY keep `get*` with a one-line note.*
- **SHOULD NOT** · name a boolean with a negation that forces double-negative reading at call sites; prefer the positive (`isEnabled` over `isNotDisabled`). *Escape hatch: use the negative only when it is the natural domain concept (`isHidden`, `isDisabled` for a control whose default is enabled); pick whichever reads cleanly at the call site and stay consistent within the module.*

```kotlin
// WRONG — "get" hides a network call and a cache mutation; reader assumes it is free
fun getUser(id: UserId): User {
    val user = httpClient.fetch(id)   // blocking I/O behind a "get"
    cache[id] = user                  // mutation behind a "get"
    return user
}

// RIGHT — the name states the cost and the effect
fun fetchAndCacheUser(id: UserId): User {
    val user = httpClient.fetch(id)
    cache[id] = user
    return user
}
```

### Units, Currency, and Scale Encoded in the Name

- **MUST** · a duration, size, rate, or money amount stored as a primitive carry its unit as a suffix (`timeoutMs`, `delaySeconds`, `payloadBytes`, `priceCents`, `ratePerSecond`; constants use `*_MS`, `*_BYTES`). *Escape hatch: when a domain type already encodes the unit (`Duration`, `Money`, `ByteCount`), the name describes the role and need not repeat it (`timeout: Duration`, not `timeoutMsDuration`).*
- **MUST** · a money amount stored as a primitive name its minor/major unit (`amountCents`, `totalDollars`) — never bare `amount`/`price` on a `number`/`Double`. *Escape hatch: the standing preference is a `Money` type outright (see Data Integrity Q13.8); the unit-suffixed primitive is the fallback when a domain type is genuinely unavailable at that layer.*
- **MUST** · a timestamp stored as a primitive name its zone or epoch basis when it is not an explicit instant type (`createdAtUtc`, `epochMillis`, `expiresAtEpochSeconds`). *Escape hatch: an explicit instant type (`Instant`, `OffsetDateTime`) already fixes the basis, so a value of that type names only its role (`createdAt: Instant`).*

```typescript
// WRONG — unit-less primitives; caller passes 30 thinking seconds, gets 30 ms
function connect(timeout: number) { socket.setTimeout(timeout) }
const price = 1999  // dollars? cents? unknowable from the name

// RIGHT — unit in the name (and ideally a domain type per Data Integrity Q13.8)
function connect(timeoutMs: number) { socket.setTimeout(timeoutMs) }
const priceCents = 1999
```

### Acronyms and Casing

- **SHOULD** · follow the **language's own acronym idiom**, consistently within a module — do not import another language's casing. This is the *judge idiomatically per language, never by importing one language's syntax into another* posture (principles.md → Simplicity) applied to casing; there is no single cross-language rule.
  - Word-casing languages (TypeScript/JS, Kotlin, Java members): treat acronyms as words — `httpUrl`, `JsonParser`, `HtmlSanitizer` (`Url` not `URL`, `Json` not `JSON`).
  - Go keeps acronyms **all-caps** by convention (golint enforces it): `ServeHTTP`, `URL`, `ID`, `JSONEncoder` — never `Url`/`Json`.
  - Swift follows its stdlib idiom — initialisms are uniformly cased, and Foundation keeps them **uppercase** (`URL`, `URLSession`, `JSONDecoder`); match that (`URL` not `Url`, `JSON` not `Json`), consistent with dependencies.md (`URLSession`, `Codable`/`JSON`).
  - C#/.NET keeps two-letter acronyms uppercase (`IO`, `DB`) and Pascal-cases longer ones (`Html`, `Json`) per the Framework Design Guidelines.
- **MAY** · in word-casing languages, two-letter acronyms be uppercase (`ID`, `UI`, `IO`, `OS`, `DB`) — apply the chosen style consistently within a module.
- **SHOULD** · keep the acronym boundary visible in whichever style the language uses; never an ambiguous run-on that hides where one acronym ends and the next begins.
- *Escape hatch: when an external contract fixes the casing (a wire field, environment variable, generated DTO, or platform API you cannot rename), match the contract exactly and isolate it at the boundary; do not propagate the foreign casing into the domain layer.*

```kotlin
// Kotlin (word-casing idiom) — WRONG: acronym shouting fragments the name
class HTTPURLClient { fun parseHTMLReport(raw: String): HtmlReport }
// RIGHT — acronyms as words; the owned return type keeps its own casing
class HttpUrlClient { fun parseHtmlReport(raw: String): HtmlReport }
```

```go
// Go (all-caps idiom) — the SAME concept; golint REQUIRES the uppercase form
type HTTPClient struct{}
func (c *HTTPClient) ParseHTMLReport(raw string) (HTMLReport, error) { /* ... */ }
// `HttpClient` / `ParseHtmlReport` would be flagged by golint — wrong for Go
```

### Scope-Proportional Length and Consistent Vocabulary

- **SHOULD** · scale a name's descriptiveness with its scope: a one-line lambda parameter MAY be `x`; an exported type MUST be fully descriptive.
- **SHOULD** · map one domain concept to one term — **SHOULD NOT** mix synonyms (`user`/`account`/`customer`) for the same entity, or verbs (`fetch`/`get`/`load`) for the same operation, within a bounded context.
- **SHOULD NOT** · encode redundant context already supplied by the enclosing type/module: in `class Order`, prefer `id` over `orderId`, `total` over `orderTotal`. *Escape hatch: yield when removing the prefix creates a genuine collision or breaks an external schema field name.*
- **SHOULD NOT** · invent abbreviations; use the full word (`message` not `msg`, `request` not `req`, `configuration` not `cfg`) unless it is a universally recognized domain term (`id`, `url`, `db`, `html`). *Escape hatch: widely understood, project-conventional short forms (`ctx`, `req`/`res` in HTTP middleware, `props` in UI components, `db`, `tx` for a transaction) are acceptable when established and consistent across the module.*

```typescript
// WRONG — three names for one concept; invented abbreviations; redundant context
class Order {
  orderId: string
  fetchUsr(req: Req): Acct { ... }   // user? account? fetch vs the get used elsewhere
}

// RIGHT — one term per concept, full words, no redundant prefix
class Order {
  id: string
  fetchCustomer(request: Request): Customer { ... }
}
```

## Self-Review Checklist

| # | Check | Required | Rule |
|---|---|---|---|
| 1 | Functions are verb phrases; pure queries are noun/adjective phrases | Required | Q3.3 |
| 2 | Types/classes/structs are noun phrases; protocols are capability nouns/adjectives | Required | Q3.3 |
| 3 | Booleans use `is`/`has`/`should`/`can`/`did`/`will`/`was` prefix (or an `all`/`any`/`every` quantifier for plural subjects) | Required | Q3.3 |
| 4 | Collections are plural nouns; maps use `<value>By<key>` | Required | Q3.3 |
| 5 | Constants UPPER_SNAKE_CASE; enum cases singular state nouns (constant doc comments are owned by documentation Q2) | Required | Q3.3 |
| 6 | File names match the primary export exactly | Required | Q3.3 |
| 7 | Throwaway-scope exemptions (loop index, lambda param, math symbol) stay within ≤5 visible lines | Required | Q3.3 |
| 8 | No banned names as primary identifiers (Manager, Helper, Util, Service/Handler/Processor-generic, Wrapper, Base, `*Impl`, Common, Factory-generic, do/run/handle) | Required | Q3.3 |
| 9 | No pronoun-grade or type-as-name locals (result, value, item, obj, info, data, tmp, list, arr, str) | Required | Q3.3 |
| 10 | Any retained banned-word compound is the genuine domain term, never the bare word | Required | Q3.3 |
| 11 | `get`/`is`/`has`/noun accessors do not mutate, do I/O, block, or throw on the normal path (memoized-getter escape hatch documented where used) | Required | Q3.3 |
| 12 | `is`/`has`/`should`/`can` return booleans; plural-noun queries return collections | Required | Q3.3 |
| 13 | Costly (I/O, O(n)+ unbounded) operations are named `fetch`/`load`/`query`/`compute`, not `get` | Required | Q3.3 |
| 14 | Booleans named positively; no double-negative call sites (negative only where it is the natural domain term) | Required | Q3.3 |
| 15 | Duration/size/rate primitives carry their unit suffix (`timeoutMs`, `payloadBytes`); unit-typed values exempt | Required | Q3.3 |
| 16 | Money primitives name their minor/major unit (`priceCents`, `totalDollars`); prefer a `Money` type (Data Integrity Q13.8) | Required | Q3.3 |
| 17 | Timestamp primitives name their zone/epoch basis (`createdAtUtc`, `epochMillis`); instant types exempt | Required | Q3.3 |
| 18 | Acronyms word-cased (`httpUrl` not `HTTPUrl`); 2-letter uppercase used consistently | Required | Q3.3 |
| 19 | Adjacent acronyms keep visible word boundaries (`xmlHttpRequest`) | Required | Q3.3 |
| 20 | Foreign/external casing matched only at the boundary, not propagated inward | Required | Q3.3 |
| 21 | Name length scales with scope; one term per concept; no synonym mixing in a context | Required | Q3.3 |
| 22 | No redundant enclosing-context prefix; no invented abbreviations (escape hatch: established short forms) | Required | Q3.3 |
