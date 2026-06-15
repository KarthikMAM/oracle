# API & Contract Design

A published API is a promise you cannot take back: every consumer you cannot see has already encoded your current behavior into their code. Designing the contract for evolution — not just for today's caller — is the difference between a surface you can grow and one you must freeze forever.

## Anchors

- **Hyrum's Law** — with enough consumers, every observable behavior of your contract becomes something someone depends on
- **Robustness Principle** (Postel, 1980) — be conservative in what you send, liberal in what you accept; temper it with strict parsing at the boundary
- **Parse, Don't Validate** (Alexis King, 2019) — turn untrusted input into a precise type once, at the edge, so the interior is total
- **Semantic Versioning** (Preston-Werner, 2013) — a version number is a machine-readable backward-compatibility contract
- **API Evolution** (Henning, *API Design Matters*, 2007) — the cost of a contract is paid by every caller, forever
- **Idempotency** (Helland — *Life Beyond Distributed Transactions*) — at-least-once delivery is reality; mutating endpoints must tolerate retries

## Q5.1 — Backward Compatibility

- **MUST** make new fields, parameters, methods, and response keys additive; removing or renaming a published element is a breaking change that **MUST** follow the Q5.3 deprecation lifecycle.
- **MUST** make new request parameters optional with a default that preserves pre-existing behavior.
- **MUST** ship new methods on a published interface/protocol with a default implementation *where the language supports defaults* (Java default methods, Kotlin interface defaults, Swift protocol extensions, Rust default trait methods). Where it does not (e.g. Go, or a Rust trait method that cannot carry a default), introduce a NEW narrow interface at the consumer instead of mutating the published one — the method addition is breaking and **MUST** follow the Q5.3 lifecycle (consumer-defined interface segregation, per type-design.md Q4.14).
- **MUST** treat narrowing an accepted input (tighter validation, smaller range, new required field) or breakingly widening an output (a new enum case a client must handle) as breaking.
- **MUST NOT** repurpose the meaning of an existing field; add a new field instead.
- *Escape hatch: a pre-1.0 or explicitly-unstable surface (annotated `@Beta`/`@Unstable`/`@experimental`, or documented as such at its declaration) MAY break incompatibly if the break is called out in release notes; the instability marker MUST be on the declaration, not only in prose.*

```kotlin
// WRONG — new required parameter breaks every existing caller
fun createOrder(items: List<Item>, couponCode: String): Order

// RIGHT — additive: optional with a default that preserves old behavior
fun createOrder(items: List<Item>, couponCode: String? = null): Order
```

## Q5.2 — Versioning Strategy

- **MUST** version public surfaces by SemVer: MAJOR for breaking, MINOR for additive, PATCH for no-contract fixes.
- **SHOULD** serve exactly one contract version per endpoint/handler, routing to a version-specific adapter at the boundary rather than threading a version flag through business logic. *Escape hatch: a single narrowly-scoped conditional at the boundary that selects the adapter is fine — document the boundary seam.*
- **MUST** make the version explicit on the wire for network APIs — a URI segment (`/v2/orders`), a media type (`application/vnd.acme.order.v2+json`), or a required header; implicit "latest" routing **MUST NOT** silently upgrade un-versioned callers across a MAJOR boundary.
- **MUST NOT** delete a prior MAJOR until it has completed the Q5.3 deprecation lifecycle.
- **SHOULD** prefer additive MINOR evolution over a MAJOR bump where a default-bearing addition (Q5.1) achieves the goal. *Escape hatch: when the only correct fix genuinely changes existing behavior, ship the MAJOR — do not contort the contract to fake compatibility.*

```typescript
// WRONG — version flag threaded through business logic, grows unboundedly
function handleOrder(req: OrderRequest, version: number): OrderResponse {
  const total = version >= 2 ? computeTotalV2(req) : computeTotalV1(req)
  return version >= 2 ? shapeV2(total) : shapeV1(total)
}

// RIGHT — route to a version-specific adapter at the boundary; core logic is version-free
const ORDER_ADAPTERS: Record<ApiVersion, OrderAdapter> = { v1: v1OrderAdapter, v2: v2OrderAdapter }
function handleOrder(req: RawRequest): OrderResponse {
  const adapter = ORDER_ADAPTERS[req.apiVersion]
  return adapter.toResponse(coreCreateOrder(adapter.toDomain(req)))
}
```

## Q5.3 — Deprecation Lifecycle

- **MUST NOT** remove a published element without completing three stages in order: **warn** (mark deprecated, keep working, point to the replacement), **error** (present but signals failure on use behind a clear opt-in/opt-out), then **remove**.
- **MUST** carry, at the declaration site: the version deprecated in, the replacement, and the earliest version removal MAY occur.
- **MUST** make the deprecation machine-detectable via the language's native annotation (`@Deprecated`, `@available(*, deprecated:)`, `@deprecated` JSDoc, `warnings.warn(DeprecationWarning)`), not a comment.
- **MUST** signal deprecation on the wire for network contracts (e.g. a `Deprecation`/`Sunset` response header) in addition to documentation.
- **MUST NOT** shorten a published removal date; you MAY extend it.

```kotlin
// RIGHT — warn stage: native annotation, deprecated-in version, replacement, and removal window all stated
@Deprecated(
  message = "Deprecated in v3.2.0. Ambiguous across time zones. Use createOrderAt(instant) instead.",
  replaceWith = ReplaceWith("createOrderAt(instant)"),
  level = DeprecationLevel.WARNING, // deprecated in v3.2.0 -> ERROR next major -> removed in v4.0.0
)
fun createOrder(localDate: LocalDate): Order
```

## Q5.4 — Wire & Serialization Compatibility

- **MUST NOT** remove a persisted/transmitted field or change its type in place; add a new field and migrate.
- **MUST** keep field identity stable: for tag-based formats (Protobuf, Thrift, FlatBuffers) tags **MUST NOT** be reused or renumbered and removed tags **MUST** be reserved; for name-based formats (JSON) the key is the identity.
- **MUST** tolerate unknown fields (forward compatibility) and **MUST** supply a safe default for absent fields (backward compatibility).
- **MUST** reserve an explicit `UNKNOWN`/`UNSPECIFIED` enum member at the zero/default position, and decoders **MUST** map unrecognized values to it rather than throwing.
- **MUST** ship any non-additive change (retype, unit change, structural reshape) with a forward migration AND keep the old representation readable until every writer of the old form is retired.
- *Escape hatch: an internal, single-deployment, never-persisted message between two services released together as one unit MAY skip reserved-tag discipline; the moment a third consumer or any persistence appears, full discipline applies.*

```protobuf
// WRONG — reusing/renumbering a tag silently corrupts every record written under the old meaning
message User {
  string email = 2;   // tag 2 previously meant `phone` — old records now mis-decode
}

// RIGHT — reserve the retired tag and name; new field gets a fresh tag; enum has a safe zero value
message User {
  reserved 2;
  reserved "phone";
  string email = 5;
  Status status = 6;
}

enum Status {
  STATUS_UNSPECIFIED = 0; // safe default for old readers + unrecognized new values
  STATUS_ACTIVE = 1;
}
```

## Q5.5 — Idempotency of Mutations

This section governs the *API-surface contract* for idempotency — how the key is accepted, returned, and signaled. The persistence and atomicity rules (the key check and the effect committed in one transaction, replay returning the stored result) are owned by the data-integrity standard; see data-integrity.md Q13.1, and follow it for the storage side.

- **MUST** accept a client-supplied idempotency key on every mutating endpoint (create/charge/transfer/send) that is not naturally idempotent, and **MUST** return the original result for a repeated key. (Dedupe committed per data-integrity.md Q13.1.)
- **MUST** reject same key + a conflicting request payload on the surface (e.g. `409 Conflict`), never silently apply it.
- **MUST** keep naturally idempotent verbs idempotent: `PUT`/`DELETE` and full-replacement updates **MUST** converge to the same state on replay.
- **MUST** keep read endpoints free of observable side effects (no state-changing writes, no consuming a one-time token).
- *Escape hatch: an operation provably idempotent by construction (e.g. a deterministic upsert keyed entirely on caller-supplied identity) MAY omit a separate key; the proof MUST be documented at the declaration.*

```typescript
// WRONG — retry/double-submit charges the customer twice
async function charge(req: ChargeRequest): Promise<Receipt> {
  return paymentGateway.capture(req.amount) // no key, no dedupe
}

// RIGHT — the surface contract as an error-as-value (no throw in business logic, per
// error-handling.md Q7.1): a key reused with a different payload is a `conflict` arm the
// caller branches on, not a thrown exception. The HTTP edge maps `conflict` → 409, `ok` → 200.
// (Storage side — the dedupe transaction and capture-outside-tx ordering — is owned by
// data-integrity.md Q13.1/Q13.2.)
type ChargeOutcome =
  | { ok: true; receipt: Receipt }      // new capture, or a replay returning the original
  | { ok: false; conflict: string }     // same key, different request payload

async function charge(req: ChargeRequest, idempotencyKey: string): Promise<ChargeOutcome> {
  const prior = await charges.find(idempotencyKey)

  if (prior && prior.requestHash !== hash(req)) return { ok: false, conflict: idempotencyKey }
  if (prior) return { ok: true, receipt: prior.receipt }   // replay: original receipt, no re-capture

  const receipt = await charges.captureAndRecord(req, idempotencyKey)   // first time; dedupe per Q13.1
  return { ok: true, receipt }
}
```

## Q5.6 — Pagination of Unbounded Collections

The *resource-bounding* mechanics — that unbounded reads are forbidden, and why cursor paging beats `OFFSET` — are owned by the performance standard; see performance.md Q11.3. This section governs the *contract shape* of pagination across the API surface.

- **MUST** paginate any endpoint or method returning a collection whose size is not statically bounded, per performance.md Q11.3, exposing pagination explicitly (a page-size parameter and a "next" signal).
- **SHOULD** use cursor/token-based pagination rather than numeric offset (rationale in performance.md Q11.3). *Escape hatch: a small, stable, infrequently-changing lookup set MAY use offset paging; document why mutation-skew is acceptable.*
- **MUST** enforce a maximum page size and **MUST** clamp (not reject) a client request above it.
- **SHOULD** make the pagination contract uniform across the whole surface — one cursor shape, one page-size parameter name, one "has more" signal.
- **MUST** treat an opaque cursor as opaque on the client side; the server **MUST** be free to change its internal encoding without a version bump (Hyrum's Law).

```python
# WRONG — unbounded result set; fine in tests, fatal in production
def list_events(self) -> list[Event]:
    return self._repo.all()  # could be millions

# RIGHT — cursor-based page with a server-enforced, clamped limit and a uniform shape
def list_events(self, page: PageRequest) -> Page[Event]:
    # Clamp the page size server-side — never trust the client's number, and clamp, don't reject.
    limit = min(page.limit, MAX_PAGE_SIZE)

    # Over-fetch one row to learn whether a next page exists, without a second COUNT query.
    rows = self._repo.after(cursor=page.cursor, limit=limit + 1)
    has_more = len(rows) > limit
    items = rows[:limit]

    next_cursor = encode_cursor(items[-1]) if has_more else None
    return Page(items=items, next_cursor=next_cursor)
```

## Q5.7 — Explicit Error Contracts

- **MUST** document every public operation's error contract: the distinct, typed failure conditions a caller can encounter and how each is signaled (typed error, error response shape, or status code).
- **MUST** make error responses typed and machine-readable — a stable error `code` enum plus a human-readable `message`; callers branch on the code without string-matching the message.
- **MUST NOT** leak internal detail across the boundary (stack traces, SQL, file paths, internal hostnames, dependency identifiers, raw upstream errors); map to a domain error at the boundary (see Q5.8 and the error-handling standard). Leaking internals is BLOCK-severity for the safety reviewer lens (also enforced by error-handling.md Q7.9).
- **MUST** evolve the error code set as a published contract: adding a code is additive; removing or repurposing one **MUST** follow the Q5.3 deprecation lifecycle.
- **SHOULD** return a correlation/request ID with every error so the caller can report it and operators can trace it. *Escape hatch: omit only where the surface has no operational logging to correlate against.*

```typescript
// WRONG — leaks internals; untyped; caller must string-match to react
res.status(500).send(err.stack) // SQL, paths, dependency names exposed

// RIGHT — typed, stable code; safe message; correlation id; no internals
type ApiError = {
  code: ErrorCode          // stable enum the caller switches on
  message: string          // human-readable, safe to display
  correlationId: string    // for support + tracing, leaks nothing
}
res.status(statusFor(code)).json({ code, message: safeMessage(code), correlationId })
```

## Q5.8 — Surface Consistency & Total Boundaries

- **SHOULD** keep naming and shape consistent across the entire surface: same concept → same name, same operation kind → same verb, uniform collection/page/error/timestamp shapes. (Project-wide naming taxonomy is owned by naming.md Q3.3; this rule is the API-surface application of it.)
- **SHOULD** expose identifiers as opaque strings to the caller, even when internally numeric, so the internal representation can change without a contract break. *Escape hatch: a value whose numeric semantics are genuinely part of the contract (a quantity, a version counter) stays numeric.*
- **MUST** make boundary functions total: every receivable input maps to a typed success or a typed, documented error — never an unchecked crash, an undocumented `null`, or undefined behavior. The parsed boundary type itself (Parse-Don't-Validate, illegal states unrepresentable) is owned by type-design.md Q4.8; this rule is its API-surface obligation.
- **MUST** distinguish optional/absent from present-but-empty deliberately and document it; conflating "no value" with "empty value" at the boundary is a contract ambiguity.

```typescript
// WRONG — partial boundary: throws on bad input, returns a raw bag callers must re-validate
function parseRequest(body: unknown): { email: string; age: number } {
  return body as { email: string; age: number } // illegal states flow inward
}

// RIGHT — total boundary: parse once into a precise type or a typed error
type ParseResult<T> = { ok: true; value: T } | { ok: false; error: ApiError }

function parseCreateUser(body: unknown): ParseResult<CreateUser> {
  const r = CreateUserSchema.safeParse(body) // e.g. zod — validates + narrows
  return r.success
    ? { ok: true, value: r.data }            // interior is total from here on
    : { ok: false, error: toApiError(r.error) }
}
```

## Self-Review Checklist

| # | Check | Required | Rule |
|---|---|---|---|
| 1 | New fields/params/methods/keys are additive only | Required | Q5.1 |
| 2 | New params optional with a behavior-preserving default | Required | Q5.1 |
| 3 | New interface/protocol methods ship with a default implementation | Required | Q5.1 |
| 4 | No existing field repurposed; input not narrowed, output not widened breakingly | Required | Q5.1 |
| 5 | Unstable/pre-1.0 breaks are marked on the declaration and noted in release notes | Required | Q5.1 |
| 6 | SemVer honored — breaking change = MAJOR bump | Required | Q5.2 |
| 7 | Version explicit on the wire; one contract version per handler | Required | Q5.2 |
| 8 | No version flags threaded through business logic — adapter at the boundary | Required | Q5.2 |
| 9 | Prior MAJOR kept until deprecation lifecycle completes | Required | Q5.2 |
| 10 | Additive MINOR preferred over MAJOR when a default-bearing addition achieves the goal | Required | Q5.2 |
| 11 | Removal follows warn -> error -> remove; never abrupt | Required | Q5.3 |
| 12 | Deprecation states version, replacement, and earliest-removal version | Required | Q5.3 |
| 13 | Deprecation is machine-detectable (native annotation / Sunset header) | Required | Q5.3 |
| 14 | Published removal date never shortened | Required | Q5.3 |
| 15 | No wire field removed or retyped in place; tags/keys stable, retired tags reserved | Required | Q5.4 |
| 16 | Decoders tolerate unknown fields and default absent fields | Required | Q5.4 |
| 17 | Wire enums reserve UNKNOWN/UNSPECIFIED; decoders map unrecognized values to it | Required | Q5.4 |
| 18 | Non-additive wire change ships with migration; old form stays readable | Required | Q5.4 |
| 19 | Mutating endpoints accept an idempotency key; repeats return the original result (dedupe persisted per data-integrity.md Q13.1) | Required | Q5.5 |
| 20 | Same key + conflicting payload rejected on the surface (409), not silently applied | Required | Q5.5 |
| 21 | Naturally idempotent verbs converge on replay; reads have no side effects | Required | Q5.5 |
| 22 | Unbounded collections are paginated (per performance.md Q11.3) | Required | Q5.6 |
| 23 | Cursor-based pagination preferred; server clamps a max page size | Required | Q5.6 |
| 24 | Uniform pagination contract; cursors opaque to clients | Required | Q5.6 |
| 25 | Every public operation documents its typed error contract | Required | Q5.7 |
| 26 | Error responses carry a stable code + safe message; no internals leaked | Required | Q5.7 |
| 27 | Error code set evolves additively / via deprecation; correlation id returned | Required | Q5.7 |
| 28 | Naming and shape consistent across the whole surface | Required | Q5.8 |
| 29 | Identifiers opaque unless numeric semantics are part of the contract | Required | Q5.8 |
| 30 | Boundary functions total — parse untrusted input into a precise type at the edge | Required | Q5.8 |
| 31 | Absent vs. empty distinguished and documented | Required | Q5.8 |
