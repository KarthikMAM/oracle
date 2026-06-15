# Performance

Performance defects are paid for in production: a quadratic loop that was fine on 100 test rows melts a CPU on 100k production rows; an unbounded fetch that fit a demo dataset OOMs the fleet at scale. These rules catch the defects that survive review precisely because they are invisible on small inputs. They do not license premature optimization — cold-path micro-optimization stays forbidden until measured.

## Anchors

- **Knuth** (*Structured Programming with go to Statements*, 1974) — "premature optimization is the root of all evil; yet we should not pass up our opportunities in that critical 3%." Optimize the measured hot path; leave the cold path readable.
- **Big-O Awareness** (Cormen et al., *Introduction to Algorithms*) — complexity is a property of the input domain, not the test fixture; nested iteration over unbounded input is a latent outage.
- **Mechanical Sympathy** (Thompson, after Jackie Stewart) — write code that works *with* the cache, allocator, and scheduler, not against them; per-frame allocation and pointer chasing are the common offenders.
- **Latency Numbers Every Programmer Should Know** (Dean, 2009) — a same-datacenter network round trip is ~10³–10⁴× a main-memory read, an intercontinental one ~10⁶×; either way, N+1 round trips, not CPU, dominate most real latency.
- **The Tail at Scale** (Dean & Barroso, 2013) — p99 latency, not the mean, defines user experience; one unbounded operation in a hot path poisons the tail, and unbounded fan-out turns one slow client into a self-inflicted denial of service.
- **Regular Expression Matching Can Be Simple And Fast** (Cox, 2007) — backtracking engines are exponential on adversarial input; untrusted regex over untrusted text is a denial-of-service primitive.
- **Amdahl's Law** (Amdahl, 1967) — the speedup from parallelism is capped by the serial fraction; parallelizing the cheap part while a serial round trip dominates buys nothing and adds risk.

## Q11 — General Rules

- **MUST NOT** perform synchronous I/O (disk, network, IPC, database, `localStorage`, keychain, large file reads) on the main/UI thread, nor CPU-bound work exceeding one frame budget (16ms@60Hz; 8ms@120Hz). · BLOCK-severity.
- **MUST NOT** dispatch to or assume the main thread in service-layer or library code — the caller chooses the observation context (see Concurrency Q8).
- **MUST NOT** nest a loop over two independently unbounded inputs (O(n·m) or worse) when a hash join, set membership, or single-pass algorithm achieves the same result; a membership/de-dup test inside a loop MUST use a hash/set, not a linear `contains`/`indexOf`/`includes` scan (see Q11.11). · BLOCK-severity.
- **MUST** bound every read of an externally sized collection (DB table, paginated API, queue, user-supplied list) — pagination, `LIMIT`, streaming cursor, or documented cap. · BLOCK-severity.
- **MUST** bound every cache, queue, buffer, and accumulator per Q11.4 (same invariant in Resource Management Q12.4, which extends it to bound-behavior semantics for all accumulating structures). · BLOCK-severity.
- **MUST NOT** apply a regex with catastrophic-backtracking constructs (nested quantifiers, overlapping alternations) to externally controlled input; see Q11.9. · BLOCK-severity (DoS vector).
- **MUST** bound fan-out concurrency over an externally sized collection — one task per element of an unbounded input is an unbounded-resource defect; see Q11.15. · BLOCK-severity.
- **MUST** justify optimizations beyond the always-wrong patterns named here (Q11.1–Q11.15) by measurement (benchmark, profile, or production metric attached to the change); **MUST NOT** merge speculative micro-optimization that trades readability for unmeasured speed.

## Q11.1 — Main Thread Discipline

- **MUST** move work whose duration scales with input off-main; observe the result on main. *Escape hatch: provably constant, trivially small work (e.g. a fixed three-field struct copy) is exempt with a one-line comment stating why the work is bounded and small — "small in practice" over an externally sized input is not constant.*

```kotlin
// WRONG — synchronous disk read on the main thread freezes the UI
fun onCreate() {
    val config = File(path).readText()   // blocking I/O on main -> ANR
    render(parse(config))
}

// RIGHT — move blocking work off-main; observe the result on main
// withContext(Dispatchers.IO) moves the blocking read off the main thread;
// the result is observed back on main (Concurrency Q8.7)
fun onCreate() {
    viewModelScope.launch {
        val config = withContext(Dispatchers.IO) { File(path).readText() }
        render(parse(config))   // back on main, cheap
    }
}
```

## Q11.2 — No N+1 Round Trips

- **MUST NOT** issue a per-element remote or out-of-process call in a loop (HTTP, RPC, GraphQL resolver, cache `get`, message publish, `stat`/`open`, IPC, DB query), including the awaited form `for (const x of xs) await remote(x)` — batch, join, or pre-fetch; if the calls are independent, bound the concurrency per Q11.15. *Escape hatch: a provably small, fixed collection (≤ a stated compile-time constant, e.g. a 3-element enum) AND no batch API exists, recorded in an inline comment — "usually small"/"product says it won't grow"/a default page size are not bounds.*

```typescript
// WRONG — N+1 (1 query for users + N queries for orders)
const users = await db.query("SELECT * FROM users WHERE active = true")
for (const user of users) {
    user.orders = await db.query("SELECT * FROM orders WHERE user_id = ?", [user.id])
}

// RIGHT — batch query (2 queries total), then group in memory
const users = await db.query("SELECT * FROM users WHERE active = true LIMIT 1000")
const userIds = users.map(u => u.id)
// One placeholder per id — a single `?` does NOT expand an array into IN (1,2,3)
// on the prepared-statement drivers (pg, better-sqlite3, JDBC, mysql2); generate them.
const placeholders = userIds.map(() => "?").join(",")
const orders = await db.query(
    `SELECT * FROM orders WHERE user_id IN (${placeholders}) LIMIT 50000`,  // bound the read (Q11.3)
    userIds,
)
const ordersByUser = groupBy(orders, o => o.userId)
for (const user of users) {
    user.orders = ordersByUser.get(user.id) ?? []
}
```

## Q11.3 — Bounded Fetch and Pagination

- **MUST** cap every read whose source size is not bounded by code you control. *Escape hatch: a read may be unbounded only when a hard upstream constraint caps the row count (e.g. a unique index on a low-cardinality enum, or a `CHECK` constraint), cited in a comment.*
- **SHOULD NOT** use `OFFSET` paging beyond a small, fixed first few pages — it re-scans and discards `n` rows per page (O(n²) across a full walk) and skips/duplicates rows under concurrent writes; use keyset/cursor paging when the table can grow large. *Escape hatch: a hard, documented cap on total pages, e.g. a UI showing only pages 1–10.*

```sql
-- WRONG — materializes the entire table; fine at 1k rows, OOM at 10M
SELECT * FROM events WHERE tenant_id = ?

-- RIGHT — keyset pagination with an explicit page size
SELECT * FROM events
WHERE tenant_id = ? AND id > ?      -- cursor
ORDER BY id
LIMIT 500                            -- bounded page; caller loops on the cursor
```

## Q11.4 — Bounded Collections

- **MUST** make every cache, queue, buffer, batch accumulator, and in-memory index bounded and name its eviction/rejection policy (LRU, LFU, FIFO, TTL, size-based, or reject-at-bound) and behavior at bound (oldest evicted, newest rejected, producer blocked, or load shed). · BLOCK-severity if unbounded on an unbounded input domain.
- **MUST** carry a size or TTL bound on a cache keyed by unbounded-cardinality key (raw user string, request id, free-form tuple) — *no escape hatch for "the key space is small in practice."*
- **SHOULD** justify the numeric bound with a one-line rationale (heap budget, working-set estimate, downstream limit, or load test) — any value is acceptable; the MUST is that the structure is bounded at all.

```kotlin
// WRONG — unbounded map; one entry per unique key grows until OOM
private val cache = mutableMapOf<String, User>()

// RIGHT — bounded LRU + TTL; max size, eviction policy, bound behavior all declared
// Bound rationale: 10k * ~2KB ≈ 20MB, well under the 256MB heap budget
private val cache: Cache<String, User> =
    Caffeine.newBuilder()
        .maximumSize(10_000)
        .expireAfterWrite(5, TimeUnit.MINUTES)
        .build()   // at bound: evict LRU, never block
```

## Q11.5 — Hoist Loop-Invariant Work

- **SHOULD** hoist expensive invariant work (compilation, parsing, reflection, allocation, I/O, cryptographic setup, or any call costing more than a few instructions) outside the loop. *Escape hatch: leave it inline when the value genuinely varies per iteration or when hoisting would extend a lock/resource scope incorrectly.*
- **MUST** treat loop-invariant work that issues a remote/out-of-process call as an N+1 governed by Q11.2.

```kotlin
// WRONG — Regex compiled on every iteration (O(n) compilations)
for (item in items) {
    if (Regex("^prefix-").matches(item.name)) {
        process(item)
    }
}

// RIGHT — compiled once, reused (1 compilation)
val prefixRegex = Regex("^prefix-")
for (item in items) {
    if (prefixRegex.matches(item.name)) {
        process(item)
    }
}
```

## Q11.6 — Batch DOM/Layout Reads and Writes

- **SHOULD** batch all layout reads (`offsetWidth`, `getBoundingClientRect`), then all writes, when the element count scales with data — interleaving forces synchronous reflow per read ("layout thrashing"). *Escape hatch: a fixed, small, known set of elements where a single interleaved read/write is clearer and provably constant-cost.*

```javascript
// WRONG — read-write-read-write thrashes layout (forced reflow per element)
elements.forEach(el => {
    const width = el.offsetWidth      // forces layout
    el.style.width = `${width * 2}px` // invalidates layout
    const height = el.offsetHeight    // forces layout AGAIN
})

// RIGHT — batch reads, then batch writes (one reflow)
const measurements = elements.map(el => ({
    width: el.offsetWidth,
    height: el.offsetHeight
}))
elements.forEach((el, i) => {
    el.style.width = `${measurements[i].width * 2}px`
    el.style.height = `${measurements[i].height * 2}px`
})
```

## Q11.7 — Parallelize Independent Awaits

- **SHOULD** run independent async work concurrently when each await carries non-trivial latency. *Escape hatch: a genuine data dependency (B needs A's result) or a deliberate concurrency limit (rate-limited downstream, ordered side effects), named in a comment.*
- **SHOULD**, in JS/TS, use `Promise.allSettled` over `Promise.all` to collect every result instead of rejecting on the first failure (JS promises are not cancellable — siblings keep running either way; the distinction is whether you receive all results). In structured concurrency, use an isolating group (`supervisorScope` / a non-throwing task group) when one child's failure should not cancel its siblings.
- **MUST** bound the fan-out per Q11.15 when the parallel work is one task per element of a collection — `Promise.all` over an unbounded array is a defect, not a fix; this rule covers a fixed, small set of distinct operations.

```typescript
// WRONG — sequential; total latency = sum of all three
const user = await fetchUser(id)
const posts = await fetchPosts(id)
const comments = await fetchComments(id)

// RIGHT — concurrent; total latency = max of the three
const [user, posts, comments] = await Promise.all([
    fetchUser(id),
    fetchPosts(id),
    fetchComments(id),
])
```

## Q11.8 — Hot-Path Allocation and String Building

- **MUST NOT** build a result by repeated copy-on-grow inside a loop — `String +=` (O(n²)), `list = list + [x]`, array spread (`acc = [...acc, x]`), or immutable-collection re-creation each re-copies the whole accumulator; append to a mutable buffer (or pre-size and fill) instead. · MUST NOT regardless of path.
- **SHOULD** avoid hot-path allocation and autoboxing (pre-size, reuse, use a builder, primitive-specialized containers). *Escape hatch: on a cold path, write the clearest code and do not pre-size or pool.*

```java
// WRONG — String += in a loop is O(n²); each concat copies the whole accumulator
String csv = "";
for (Row r : rows) {
    csv += r.toCsv() + "\n";   // reallocates and copies every iteration
}

// RIGHT — StringBuilder is O(n), pre-sized to avoid intermediate growth
StringBuilder sb = new StringBuilder(rows.size() * 64);
for (Row r : rows) {
    sb.append(r.toCsv()).append('\n');
}
String csv = sb.toString();
```

## Q11.9 — Safe Regular Expressions on Untrusted Input

- **MUST**, for any regex over externally controlled input, be free of nested/overlapping quantifiers OR length-cap the input before matching.
- **MUST** run a regex pattern itself supplied or influenced by an external party (user-defined search, config-driven filter) on a linear-time engine (RE2-class) or under a wall-clock timeout — a "safe-looking" pattern is not enough when the pattern is untrusted. *Escape hatch for fully trusted, code-controlled input: a comment asserting the input is not externally influenced.*

```typescript
// WRONG — nested quantifier; "(a+)+$" backtracks exponentially on "aaaa...!"
const re = /^(\w+\s?)*$/
if (re.test(userInput)) accept()        // hangs the event loop on adversarial input

// RIGHT — anchored linear pattern (each char consumed once), or cap input length first
const re = /^\w+(?:\s\w+)*$/             // no nested quantifier; linear time
if (re.test(userInput)) accept()
```

## Q11.10 — Measurement Discipline

- **MUST** fix the always-wrong patterns in Q11.1–Q11.9 and Q11.11–Q11.15 regardless of measurement.
- **MUST** attach evidence (benchmark, profiler output, or production metric showing the path is hot and the change helps) to any *other* hot-path optimization; **MUST NOT** merge unmeasured micro-optimization.
- **SHOULD** let clarity win over speed on cold paths — *the escape hatch from every SHOULD-strength performance rule above is "this is a cold path; the clearer form is chosen deliberately."*
- **MUST NOT** introduce premature optimization that obscures intent (manual loop unrolling, bit-twiddling, caching that adds invalidation risk) on an unmeasured path.

## Q11.11 — Right Data Structure for the Access Pattern

- **MUST** use a hash/set membership test (O(1)), not a linear scan, when both the scanned collection and the surrounding loop are over independently growable inputs (the accidental-quadratic defect of Q11).
- **SHOULD** match the structure to the access pattern when one side is a small fixed constant — *escape hatch: a linear scan over a provably small, fixed-size collection (≤ a stated compile-time constant) is fine and often clearer, in a one-line comment; equally, do not reach for a heavyweight structure where a list suffices on a cold path.*

```typescript
// WRONG — .includes() is a linear scan; this loop is O(n·m), quadratic when both grow
const allowed = loadAllowedIds()             // array, length m
const result = incoming.filter(x => allowed.includes(x.id))   // O(n·m)

// RIGHT — Set membership is O(1); the filter is O(n)
const allowed = new Set(loadAllowedIds())    // build once
const result = incoming.filter(x => allowed.has(x.id))        // O(n)
```

## Q11.12 — No Per-Element Logging or Serialization on Hot Paths

- **MUST NOT** eagerly construct a per-element log message when the level is disabled (use lazy/parameterized logging), nor serialize a whole collection inside a loop over that collection (an accidental quadratic, see Q11.8).
- **SHOULD NOT** keep surviving per-element logging on a hot path. *Escape hatch: a genuine per-element audit/trace requirement (each element's outcome must be individually recorded for correctness or compliance), named in a one-line comment; prefer a sampled or aggregated form where the requirement allows.*

```kotlin
// WRONG — debug log + string interpolation per row; formats the message even when
// the level is disabled, and emits N log lines per request
for (row in rows) {
    log.debug("processing row ${row.id} with payload ${row.toJson()}")  // N serializations
    handle(row)
}

// RIGHT — guard the level so the message is never built when disabled; log once with a count
if (log.isDebugEnabled) log.debug("processing {} rows", rows.size)   // 1 line, lazy args
for (row in rows) {
    handle(row)
}
```

## Q11.13 — Compute Once, Reuse — No Redundant Re-Derivation

- **SHOULD** compute-once when redundant work is expensive (I/O, parse, sort, compile, reflection, cryptographic setup) and the inputs are unchanged between derivations (the loop-invariant case of Q11.5 lifted to the call/property level). *Escape hatch: do not introduce a cache whose invalidation you cannot prove correct — a stale-cache bug is worse than recomputation; when inputs can change, prefer recompute-on-change (invalidate at the write) over time-based/manual invalidation, and document the trigger.*

```python
# WRONG — re-parses and re-compiles the schema on every call
def validate(payload: dict) -> bool:
    schema = json.loads(open("schema.json").read())   # I/O + parse every call
    return Validator(schema).is_valid(payload)

# RIGHT — parse/compile once at module load; reuse the immutable validator
_VALIDATOR = Validator(json.loads(Path("schema.json").read_text()))  # one-time setup
def validate(payload: dict) -> bool:
    return _VALIDATOR.is_valid(payload)
```

## Q11.14 — Short-Circuit Before Expensive Work

- **SHOULD** order conditions so the cheap, common-case rejection runs before the expensive computation/allocation/round trip. *Escape hatch: keep the clearer order on a cold path, or when the "expensive" branch is provably cheap, with a one-line note.*
- **MUST** guard before issuing the work when the discarded work is a remote/out-of-process call or an unbounded allocation — issuing a round trip or materializing a large object only to discard it is the same N+1 / unbounded-work defect as Q11.2 and Q11.3.

```typescript
// WRONG — fetches the remote profile for every user, then discards inactive ones
for (const user of users) {
    const profile = await fetchProfile(user.id)   // round trip for every user
    if (user.isActive && profile.optedIn) notify(user, profile)
}

// RIGHT — reject on the cheap local predicate first, then BATCH the survivors'
// fetches in one round trip (no serial per-element await — that would be the
// N+1 of Q11.2; an unbounded per-element fan-out would violate Q11.15)
const active = users.filter(u => u.isActive)          // cheap filter, no I/O
const profiles = await fetchProfiles(active.map(u => u.id))  // one batched round trip
for (const user of active) {
    const profile = profiles.get(user.id)
    if (profile?.optedIn) notify(user, profile)
}
```

## Q11.15 — Bounded Fan-Out and Concurrency

- **MUST** cap the in-flight count when fanning out over an externally sized collection — unbounded fan-out is an unbounded-resource defect (the parallel counterpart to the unbounded fetch in Q11.3 and the back-pressure rule in Resource Management Q12.7). *Escape hatch from bounding entirely: a provably small, fixed collection (≤ a stated compile-time constant) may fan out fully, recorded in a comment — the Q11.7 case.*
- **SHOULD** justify the concurrency limit with a one-line rationale (downstream quota, pool size, Little's Law estimate) — any limit is acceptable.

```kotlin
// WRONG — launches a coroutine per id; unbounded concurrency floods the service
coroutineScope { ids.map { id -> async { fetchDetail(id) } }.awaitAll() }

// RIGHT — a Semaphore caps concurrent in-flight calls
// Bound rationale: 10 concurrent matches the downstream's per-client quota
val gate = Semaphore(permits = 10)
val results = coroutineScope {
    ids.map { id -> async { gate.withPermit { fetchDetail(id) } } }.awaitAll()
}
```

## Self-Review Checklist

| # | Check | Required | Rule |
|---|---|---|---|
| 1 | No synchronous I/O on the main/UI thread | Required | Q11 / Q11.1 |
| 2 | No CPU-bound work exceeding one frame budget (16ms@60Hz) on the main thread | Required | Q11 / Q11.1 |
| 3 | Externally sized work moves off-main regardless of today's typical size | Required | Q11.1 |
| 4 | Service/library code does not dispatch to or assume the main thread | Required | Q11 |
| 5 | No accidental O(n·m) nesting over independently unbounded inputs | Required | Q11 |
| 6 | No N+1 round trips; per-element remote calls (incl. serial `for...await`) batched or bound-documented | Required | Q11 / Q11.2 |
| 7 | Externally sized reads are paginated, limited, or streamed in bounded batches | Required | Q11 / Q11.3 |
| 8 | Large/growing tables use keyset pagination, not deep OFFSET paging | Required | Q11.3 |
| 9 | Every cache/queue/buffer/accumulator declares max size | Required | Q11 / Q11.4 |
| 10 | Every cache/queue/buffer declares eviction policy + behavior at bound | Required | Q11 / Q11.4 |
| 11 | Bound numbers carry a one-line rationale | Required | Q11.4 |
| 12 | Caches keyed by unbounded-cardinality keys carry a size or TTL bound | Required | Q11.4 |
| 13 | Loop-invariant expensive/remote work (compile/parse/alloc/I-O) is hoisted | Required | Q11 / Q11.5 |
| 14 | UI code batches layout reads, then writes (no thrashing) | Required | Q11.6 |
| 15 | Independent awaits run concurrently; dependencies/limits documented | Required | Q11 / Q11.7 |
| 16 | No O(n²) string concatenation or copy-on-grow accumulation in a loop; builder/buffer used | Required | Q11.8 |
| 17 | Hot-path allocation/autoboxing minimized; cold paths left clear | Required | Q11.8 |
| 18 | Regex over untrusted input is backtracking-safe or input is length-capped | Required | Q11 / Q11.9 |
| 19 | Externally supplied regex patterns run on a linear engine or under a timeout | Required | Q11.9 |
| 20 | Non-trivial hot-path optimizations carry measurement evidence | Required | Q11 / Q11.10 |
| 21 | No unmeasured micro-optimization that obscures intent | Required | Q11.10 |
| 22 | Membership/de-dup tests use a hash/set, not a linear scan, when both sides grow | Required | Q11 / Q11.11 |
| 23 | Container choice matches the access pattern (no needless map on cold paths) | Required | Q11.11 |
| 24 | No eager per-element logging when the level is disabled; no per-element collection serialization | Required | Q11.12 |
| 25 | Per-element logging/metrics on hot paths aggregated or sampled unless audit-required | Required | Q11.12 |
| 26 | Expensive pure derivations (sort/parse/compile) computed once, not re-derived per call/iteration | Required | Q11.13 |
| 27 | Caching is only introduced where invalidation is provably correct; trigger documented | Required | Q11.13 |
| 28 | Cheap rejections ordered before expensive work; no round trip/large alloc later discarded | Required | Q11.14 |
| 29 | Fan-out over an externally sized collection is concurrency-bounded | Required | Q11 / Q11.15 |
| 30 | Concurrency limits carry a one-line rationale | Required | Q11.15 |
