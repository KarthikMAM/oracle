# Data Integrity

Data outlives the process that wrote it; a single partial write, lost update, or fail-open default corrupts state that no later request can repair.

## Anchors

- **Parse, Don't Validate** (Alexis King, 2019) — turn untrusted input into a domain type at the boundary; never propagate unparsed data
- **ACID** (Härder & Reuter, 1983) — atomicity, consistency, isolation, durability as the contract for a correct write
- **Fail-Safe Defaults** (Saltzer & Schroeder, 1975) — on error or ambiguity, deny; never default to permit
- **Idempotence** (Helland — *Idempotence Is Not a Medical Condition*, 2012) — at-least-once delivery is the norm; every mutation must tolerate replay
- **Evolutionary Database Design** (Fowler & Sadalage, 2003) — schema changes are forward-compatible, reversible, and tested like code
- **A Critique of ANSI SQL Isolation Levels** (Berenson et al., 1995) — isolation level is an explicit, justified choice, not a default

## Q13.1 — Idempotent Mutations

- **MUST** make every externally triggered, state-changing operation idempotent OR deduplicate it by a caller-supplied idempotency key persisted in the same transaction as the effect, under a `UNIQUE` constraint on that key (the schema-enforced dedupe of Q13.6).
- **MUST** check the idempotency key and commit the effect atomically; a check-then-act split across two transactions **MUST NOT** be used. Atomicity alone is not isolation: under a non-serializable isolation level two concurrent replays can both see the key absent and both commit, so the in-transaction check-then-insert is only safe when the `UNIQUE` constraint exists (the losing insert fails and rolls its transaction back) OR the isolation level is `SERIALIZABLE` (Q13.4). A check-then-insert with neither still double-applies.
- **MUST** store the original response in the idempotency record so a replay returns the first result, not a fresh re-computation.
- **SHOULD** prefer naturally idempotent formulations (set-to-value, upsert by primary key, conditional write) over key-based dedupe. *Use key-based dedupe when the effect is inherently non-idempotent (charge, send, increment).*

```typescript
// WRONG — blind increment; a retried delivery double-charges the wallet
async function debit(userId: string, amount: Money): Promise<void> {
  const balance = await db.getBalance(userId)
  await db.setBalance(userId, balance.minus(amount))
}

// RIGHT — idempotency key + effect committed in one transaction.
// recordDebit writes to a table with a UNIQUE(idempotencyKey) constraint (Q13.6): if a concurrent
// replay raced past findDebit, the second recordDebit hits the constraint, throws, and rolls its
// whole transaction back — so the loser neither double-debits nor records a duplicate (Q13.4).
async function debit(userId: string, amount: Money, idempotencyKey: string): Promise<DebitResult> {
  return db.transaction(async (tx) => {
    const prior = await tx.findDebit(idempotencyKey)
    if (prior) return prior.result            // replay returns the original result
    const result = await tx.applyDebit(userId, amount)
    await tx.recordDebit(idempotencyKey, result)   // UNIQUE(idempotencyKey): a racing replay fails here and rolls back
    return result
  })
}
```

## Q13.2 — Transactional Boundaries

- **MUST** execute a logical mutation spanning multiple writes inside one transaction; partial commits that leave dangling references or non-summing balances **MUST NOT** be possible.
- **MUST NOT** enclose network calls or other slow non-transactional work inside a transaction; capture external results first, then open a short transaction.
- **MUST NOT** treat a non-transactional external side effect (publish, email, charge) inside a transaction as rolled back when the transaction aborts; use the transactional-outbox pattern — write the intent inside the transaction, dispatch after commit. The API-surface application of this for a charge (capture first, then a short transaction records the receipt against the idempotency key) is api-design.md Q5.5.
- **MUST NOT** catch a mid-transaction error and continue committing surviving writes; on any failure within the unit the entire transaction **MUST** roll back.

```kotlin
// WRONG — two independent writes; a crash between them leaves an orphaned line item
fun placeOrder(order: Order, item: LineItem) {
    orderRepo.insert(order)
    // process dies here -> order exists, item lost, totals wrong
    lineItemRepo.insert(item)
}

// RIGHT — one atomic unit; outbox row dispatched only after commit
fun placeOrder(order: Order, item: LineItem) {
    transaction {
        orderRepo.insert(order)
        lineItemRepo.insert(item)
        outbox.enqueue(OrderPlaced(order.id))   // published by a poller after commit
    }
}
```

## Q13.3 — Parse Untrusted Data at the Boundary

- **MUST** parse external data at every ingesting boundary into a domain type whose constructor rejects invalid values; validated-but-unparsed data (a `boolean isValid` check returning the raw input) **MUST NOT** be propagated.
- **MUST** reject invalid input at the boundary with a precise error; silent coercion (clamping, truncating, default-substitution for missing required fields) **MUST NOT** be used.
- **MUST** make a field required by an invariant non-optional in the domain type; optionality in the type **MUST** reflect genuine optionality in the domain.

```swift
// WRONG — raw dictionary flows inward; missing/garbage fields surface deep in the stack
func handle(_ json: [String: Any]) {
    let cents = json["amountCents"] as? Int ?? 0      // silent default hides a missing field
    process(cents)
}

// RIGHT — parse into a domain type at the boundary; reject invalid up front
struct ChargeRequest {
    let amount: Money
    var currency: Currency { amount.currency }   // derived, not stored — one source of truth (Q13.8)
    init(json: [String: Any]) throws {
        guard let cents = json["amountCents"] as? Int, cents > 0 else {
            throw ParseError.invalidField("amountCents")
        }
        guard let code = json["currency"] as? String, let currency = Currency(code: code) else {
            throw ParseError.invalidField("currency")
        }
        self.amount = Money(minorUnits: cents, currency: currency)   // Money carries its currency
    }
}
```

## Q13.4 — Concurrency Control for Racing Writes

- **MUST** guard any racing read-modify-write on shared persistent state with a concurrency-control mechanism — optimistic (version column / compare-and-set) or pessimistic (`SELECT ... FOR UPDATE`); an unguarded read-then-write **MUST NOT** be used.
- **SHOULD** default to optimistic control on low-contention paths; reserve pessimistic locking for high-contention hot rows or multi-row units where retry storms would be worse. The choice between optimistic and pessimistic control **MUST** be documented at the call site.
- **MUST** state and justify the isolation level (e.g., `READ COMMITTED` vs `SERIALIZABLE`) whenever it departs from the connection default.

```kotlin
// WRONG — lost update: two concurrent renames, last writer wins, the other vanishes
fun rename(id: Id, name: String) {
    val doc = repo.find(id)
    repo.save(doc.copy(name = name))
}

// RIGHT — optimistic concurrency; mismatched version is an expected outcome returned as a
// value (the loser re-reads and retries), not a thrown exception caught for flow (error-handling.md Q7.6).
sealed interface RenameResult {
    object Applied : RenameResult
    data class Conflict(val id: Id) : RenameResult
}
fun rename(id: Id, name: String, expectedVersion: Long): RenameResult {
    val updated = repo.update(
        id = id,
        name = name,
        whereVersion = expectedVersion,   // UPDATE ... SET version=version+1 WHERE version=?
    )
    if (updated == 0) return RenameResult.Conflict(id)   // caller branches on this and retries

    return RenameResult.Applied
}
```

## Q13.5 — Safe Schema Migrations

- **MUST NOT** run a destructive migration (drop column, drop table, narrow a type, irreversible backfill) in production without a verified restorable recovery point — a backup OR a point-in-time snapshot/PITR window — confirmed restorable to the pre-migration state and covering the change window. *Escape hatch: an existing continuous-PITR or snapshot mechanism already covering the window satisfies this; a fresh full backup taken synchronously before the change is not required when such a recovery point is in place and its restorability has been verified.*
- **MUST** make migrations forward-compatible so deployed code works against both old and new schema and deploy/migrate proceed in either order; use expand-then-contract.
- **MUST** give every migration a tested rollback (a `down` step or documented reverse procedure), exercised against production-shaped data before it ships.
- **SHOULD** defer destructive contraction (dropping the old column) to a separate, later migration after the new shape has been live and verified.

```python
# WRONG — single destructive migration; rolling back the app now reads a column that is gone
def migrate(db):
    db.execute("ALTER TABLE users RENAME COLUMN email TO email_address")  # old code breaks instantly

# RIGHT — expand/contract across releases; each step is reversible and forward-compatible
def migrate_expand(db):
    db.execute("ALTER TABLE users ADD COLUMN email_address TEXT")          # additive, old code unaffected
    db.execute("UPDATE users SET email_address = email WHERE email_address IS NULL")

def rollback_expand(db):
    db.execute("ALTER TABLE users DROP COLUMN email_address")              # tested before shipping
# A LATER release, after new code is live and verified, drops the old column with a backup in hand.
```

## Q13.6 — Fail Closed on Error

- **MUST** fail closed on error, ambiguity, or timeout in any check that gates a mutation or grants access — deny the operation, reject the write, return the safe restrictive default; failing open **MUST NOT** occur.
- **MUST** enforce referential and invariant checks at write time in the schema (foreign-key, uniqueness, check constraints); application-level checks alone are insufficient.
- **MUST** test the error path of a fail-closed check.

```typescript
// WRONG — dependency error is swallowed and treated as "permitted"; an outage opens the gate
async function canTransfer(userId: string): Promise<boolean> {
  try {
    return await fraudService.isClear(userId)
  } catch {
    return true   // FAIL-OPEN: fraud check down => everyone passes
  }
}

// RIGHT — error denies; the invariant holds even when the dependency is unavailable.
// Goes through safeRunAsync and branches on the SafeResult (no raw try/catch in business logic,
// cancellation not swallowed) — consistent with error-handling.md Q7.1/Q7.7, which reinforces this rule.
async function canTransfer(userId: string): Promise<boolean> {
  const r = await safeRunAsync(() => fraudService.isClear(userId))
  if (r.success) return r.value
  logger.error("fraud check unavailable, denying transfer", { userId, cause: r.error })
  return false  // FAIL-CLOSED
}
```

## Q13.7 — Explicit Delivery Semantics

- **MUST** state the delivery semantics of every queue, stream, and webhook consumer explicitly (in code or adjacent documentation), and write consumers to the stated semantics.
- **MUST** make at-least-once consumers idempotent per Q13.1, because duplicate delivery is guaranteed eventually.
- **MUST NOT** acknowledge a message before its effect is durably committed; process-then-ack is the safe ordering.
- **SHOULD** implement exactly-once as at-least-once delivery plus idempotent application, not rely on it as a transport guarantee.

```kotlin
// WRONG — ack before commit; a crash here drops the event entirely
fun onMessage(msg: Message) {
    queue.ack(msg)              // message gone
    repository.apply(msg)       // crash here => effect never happens, message not redelivered
}

// RIGHT — at-least-once: idempotent apply, then ack only after durable commit
fun onMessage(msg: Message) {
    repository.applyIdempotent(msg.idempotencyKey, msg.payload)  // safe to replay (Q13.1)
    queue.ack(msg)                                               // redelivered if we crash before this
}
```

## Q13.8 — Correct Types for Money, Time, and Identifiers

- **MUST** represent money as integer minor units or an arbitrary-precision decimal carrying its currency; binary floating-point (`float` / `double` / `Double`) **MUST NOT** represent money.
- **MUST** store and transport timestamps in UTC and zone-explicit; naive local-time storage **MUST NOT** be used for instants.
- **MUST** treat identifiers as opaque — never parsed for embedded meaning, never compared with locale-sensitive or case-insensitive rules unless the format guarantees it, never reformatted. **SHOULD** give each identifier a distinct domain type (not a bare `String`) so an order id cannot be passed where a user id is expected.

```typescript
// WRONG — float money, naive local time, stringly-typed ids
function total(prices: number[]): number {
  return prices.reduce((a, b) => a + b, 0)   // 0.1 + 0.2 = 0.30000000000000004
}
const createdAt = "2026-06-13 14:30:00"      // which zone? unknowable
function refund(orderId: string, userId: string) { /* easy to swap the two */ }

// RIGHT — integer minor units, UTC instant, distinct opaque id types
function total(prices: Money[]): Money {
  return prices.reduce((a, b) => a.plus(b), Money.zero("USD"))   // exact integer cents
}
const createdAt: Instant = Instant.now()      // UTC, zone-explicit
type OrderId = string & { readonly __brand: "OrderId" }
type UserId  = string & { readonly __brand: "UserId" }
function refund(orderId: OrderId, userId: UserId) { /* the compiler rejects a swap */ }
```

## Self-Review Checklist

| # | Check | Required | Rule |
|---|---|---|---|
| 1 | Every externally triggered mutation is idempotent or deduplicated by a persisted key | Required | Q13.1 |
| 2 | Idempotency key check and effect commit in one atomic transaction | Required | Q13.1 |
| 3 | Replay returns the original stored response, not a fresh recomputation | Required | Q13.1 |
| 4 | Multi-write mutations execute in a single transaction; no partial-commit corruption | Required | Q13.2 |
| 5 | No network/slow work or non-transactional side effects inside a transaction (use outbox) | Required | Q13.2 |
| 6 | No catch-and-continue that commits surviving writes mid-transaction | Required | Q13.2 |
| 7 | External data parsed into a domain type at the boundary; no validate-only propagation | Required | Q13.3 |
| 8 | Invalid input rejected at boundary; no silent coercion/clamping/default substitution | Required | Q13.3 |
| 9 | Required-by-invariant fields are non-optional in the domain type | Required | Q13.3 |
| 10 | Racing read-modify-write guarded by optimistic or pessimistic concurrency control | Required | Q13.4 |
| 11 | Isolation-level choice stated and justified when it departs from the default | Required | Q13.4 |
| 12 | Destructive migration has a verified restorable recovery point (backup or PITR snapshot) covering the change window | Required | Q13.5 |
| 13 | Migration is forward-compatible (expand/contract); deploy and migrate order-independent | Required | Q13.5 |
| 14 | Migration has a tested rollback exercised against production-shaped data | Required | Q13.5 |
| 15 | Gating checks fail closed (deny) on error/ambiguity/timeout; never fail open | Required | Q13.6 |
| 16 | Referential/invariant constraints enforced in the schema, not application-only | Required | Q13.6 |
| 17 | Fail-closed error path is tested | Required | Q13.6 |
| 18 | Delivery semantics stated explicitly; at-least-once consumers are idempotent | Required | Q13.7 |
| 19 | Message acked only after its effect is durably committed (process-then-ack) | Required | Q13.7 |
| 20 | Money uses integer minor units or arbitrary-precision decimal, never binary float | Required | Q13.8 |
| 21 | Timestamps stored/transported in UTC and zone-explicit; no naive local instants | Required | Q13.8 |
| 22 | Identifiers treated as opaque and given distinct domain types | Required | Q13.8 |
