# Resource Management

Every acquired resource is a debt that must be repaid on every exit path; leaked handles, unbounded buffers, and orphaned tasks are the quiet killers that survive code review and surface as 3 a.m. outages.

## Anchors

- **RAII — Resource Acquisition Is Initialization** (Stroustrup, 1984) — tie a resource's lifetime to a scope so release is automatic and exception-safe
- **Release It!** (Nygard, 2007) — bulkheads, timeouts, and bounded resource pools are what keep one slow dependency from cascading into total failure
- **Little's Law** (Little, 1961) — concurrency = arrival-rate × latency; every queue and pool must be sized against it or it grows without bound
- **Structured Concurrency** (Sustrik; Swift SE-0304) — a task's lifetime is bounded by its scope, so cancellation and cleanup are guaranteed, never orphaned
- **The Tail at Scale** (Dean & Barroso, 2013) — without timeouts and bounded retries, tail latency and retry storms amplify a single slow node into a fleet-wide brownout

## Q12 — General Rules

- **MUST** release every acquired resource (file handle, socket, connection, lock, stream, subscription, observer, native pointer, OS timer, temp file) on **every** exit path — normal return, early return, throw, cancellation. *SHOULD use the language's scope-bound mechanism (`try-with-resources`, `defer`, `use`, `with`, `finally`, RAII/`Drop`, `withTaskGroup`); a manual close that provably runs on every path also satisfies the MUST.*
- **MUST** make cleanup idempotent — calling `close`/`dispose`/`cancel` twice is a safe no-op.
- **MUST** give every operation that can block on something outside the current process or component boundary (network call, DB query, IPC, subprocess, a lock that can wait, queue poll, a cross-process/cross-service `await`) an explicit, documented timeout. *A purely in-process await/suspension (an in-memory actor, a local channel, a sibling `Deferred`) is out of scope — a deadline there is usually meaningless; see Q12.3, which owns this rule.*
- **MUST** declare ALL THREE for every cache, queue, buffer, batch accumulator, and in-memory index: **maximum size**, **eviction/rejection policy**, **behavior at bound** (Performance Q11.4 states the same bounded-structure invariant for caches; here it extends to every accumulating structure — see Performance Q11.4).
- **MUST** give every retry loop a finite ceiling and a retriable-error predicate. **SHOULD** use exponential backoff with jitter (see Q12.5). *Escape hatch on the strategy: a low fixed-ceiling backoff, a decorrelated-jitter scheme, or deferring to a circuit breaker is acceptable when the call site documents why; the finite ceiling and retriable predicate remain mandatory.*
- **MUST** tie every spawned task, coroutine, thread, or subscription to a lifecycle or cancellation token, and cancellation MUST propagate to children (see Concurrency Q8).
- **MUST** apply back-pressure (bounded channel, blocking submit, reject-and-shed, or rate limit) when a producer can outpace its consumer.
- **MUST** explicitly size and bound connection pools, thread pools, and semaphores, with a timed acquisition. **SHOULD** carry a one-line bound rationale. *Escape hatch on the number: any value is acceptable with that rationale; an unbounded pool or new-connection-per-request pattern has none.*

## Q12.1 — Release on Every Exit Path

- **MUST** release every acquired resource on every exit path — return, early return, throw, cancellation.
- **SHOULD** acquire with a scope-bound construct so the compiler or runtime guarantees release. *A manual close that provably runs on every path is also correct.*

```java
// WRONG — leaks the stream if read() throws
InputStream in = url.openStream();
byte[] data = in.readAllBytes();   // throws -> close() never runs
in.close();

// RIGHT — try-with-resources closes on every path, including exceptions
try (InputStream in = url.openStream()) {
    return in.readAllBytes();
}
```

In Rust, `Drop` runs the destructor at end of scope on every path including `panic` unwinding; an explicit `close()` is unnecessary. Do not `mem::forget` a resource guard.

## Q12.2 — Idempotent Cleanup

- **MUST** tolerate being called more than once and after partial initialization (cleanup runs from explicit close, error unwinding, cancellation, and deinit).

```typescript
// WRONG — double-dispose throws "already closed"; cancellation + finally both fire
class Connection {
  dispose(): void {
    this.socket.close()   // second call throws
  }
}

// RIGHT — guard makes dispose a safe no-op on repeat calls
class Connection {
  private closed = false
  dispose(): void {
    if (this.closed) return
    this.closed = true
    this.socket.close()
  }
}
```

## Q12.3 — Timeouts on Every External Operation

- **MUST** bound every external operation by some timeout; an unbounded blocking call is BLOCK-severity.
- **SHOULD** document a one-line rationale for the timeout value (upstream p99 plus buffer, downstream SLA, or load-test result). *Any value is acceptable with that rationale.*

```kotlin
// WRONG — no timeout; a hung dependency holds this coroutine and its connection forever
val result = remoteService.fetch(id)

// RIGHT — explicit deadline; documented why this value
// 2s p99 upstream + 500ms buffer; caller surfaces timeout as a retriable failure
val result = withTimeout(2_500.milliseconds) {
    remoteService.fetch(id)
}
```

```go
// RIGHT — context deadline propagates the timeout into the call and cancels it
ctx, cancel := context.WithTimeout(ctx, 2500*time.Millisecond)
defer cancel()   // release the context's timer on every path
return service.Fetch(ctx, id)
```

## Q12.4 — Bounded Caches, Queues, and Buffers

- **MUST** declare maximum size, eviction/rejection policy, and behavior at the bound; unbounded growth on an unbounded input domain is BLOCK-severity.
- **SHOULD** carry a one-line rationale for the numeric bound. *Any size is acceptable with that rationale.*

```kotlin
// WRONG — unbounded map; one cache key per unique input id grows without limit
private val cache = mutableMapOf<String, User>()

// RIGHT — bounded LRU: max 10k entries, evict least-recently-used, never blocks
// Bound rationale: 10k * ~2KB ≈ 20MB ceiling, well under the 256MB heap budget
private val cache: Cache<String, User> =
    Caffeine.newBuilder()
        .maximumSize(10_000)
        .expireAfterWrite(Duration.ofMinutes(5))
        .build()
```

## Q12.5 — Bounded Retries With Backoff and Jitter

- **MUST** give the retry loop a finite ceiling and gate it on a retriable-error predicate.
- **SHOULD** back off exponentially with jitter, and **SHOULD** live in a dedicated wrapper rather than inline (see Error Handling Q7). *Escape hatch on the strategy: a low fixed-ceiling backoff, a decorrelated-jitter scheme, or deferring to a circuit breaker is acceptable when the call site documents why; the ceiling and predicate stay mandatory.*

```kotlin
// WRONG — infinite, fixed-interval retry retries non-retriable errors and synchronizes the fleet
while (true) {
    try { return call() } catch (e: Exception) { Thread.sleep(1_000) }
}

// RIGHT — capped attempts, exponential backoff, full jitter, retriable predicate.
// The wrapper itself honors the Error Handling Q7.8 carve-outs, independent of the
// predicate: cancellation is re-thrown first, and catching Exception (not Throwable)
// lets JVM Error (OOM/StackOverflow) propagate. A cancelled coroutine unwinds, never retries.
suspend fun <T> withRetry(
    maxAttempts: Int = 4,
    base: Duration = 100.milliseconds,
    isRetriable: (Throwable) -> Boolean,
    block: suspend () -> T,
): T {
    var attempt = 0
    while (true) {
        try {
            return block()
        } catch (e: CancellationException) {
            throw e                                              // never retry cancellation (Q7.8 / Q12.6)
        } catch (e: Exception) {                                 // Exception, not Throwable: let fatal Error propagate
            if (++attempt >= maxAttempts || !isRetriable(e)) throw e
            val exp = minOf(attempt - 1, 30)                     // cap the shift so a large maxAttempts can't overflow
            val ceiling = base * (1L shl exp)                    // exponential
            // full jitter over [0, ceiling); a 0 draw (immediate retry) is intentional and in-spec
            val delayMs = Random.nextLong(0, ceiling.inWholeMilliseconds)
            delay(delayMs.milliseconds)
        }
    }
}
```

## Q12.6 — Cancellation Propagation, No Orphans

- **MUST** cancel children when a parent is cancelled or fails; an orphaned task holds memory, mutates dead state, and writes to a closed sink.
- **MUST** cooperatively check cancellation in CPU-bound or polling loops (`Task.checkCancellation()`, `ctx.Err()`, `ensureActive()`).

```kotlin
// WRONG — application-global task outlives its owner; cancelling the owner orphans it
fun load() {
    GlobalScope.launch { view.render(repository.fetch()) }  // view may be destroyed
}

// RIGHT — lifecycle-bound scope; owner cancellation cancels the child
fun load() {
    viewModelScope.launch { view.render(repository.fetch()) }
}
```

```swift
// WRONG — detached task with no stored handle can never be cancelled
func start() {
    Task.detached { await self.poll() }   // orphan on deinit
}

// RIGHT — store the handle, capture self weakly so the running task does not retain
// the owner (a strong capture would keep deinit from ever firing), and check cancellation
// each iteration so the task actually stops (Concurrency Q8.1 / Q8.5)
private var pollTask: Task<Void, Never>?
func start() {
    pollTask = Task { [weak self] in
        while !Task.isCancelled {
            guard let self else { return }
            await self.pollOnce()
        }
    }
}
deinit {
    pollTask?.cancel()
}
```

## Q12.7 — Back-Pressure on Fast Producers

- **MUST** apply some back-pressure rather than an unbounded buffer when a producer can outrun its consumer; the unbounded-buffer alternative is an out-of-memory crash.
- **SHOULD** document which overflow policy (suspend, drop-oldest, drop-newest, reject-and-shed) was chosen and why. *Any policy is acceptable with that documentation.*

```kotlin
// WRONG — UNLIMITED channel buffers without bound; fast producer OOMs the process
val channel = Channel<Event>(Channel.UNLIMITED)

// RIGHT — bounded buffer; SUSPEND back-pressures the producer at capacity
// Bound rationale: 64 in-flight events matches the consumer's batch size
val channel = Channel<Event>(capacity = 64, onBufferOverflow = BufferOverflow.SUSPEND)
```

## Q12.8 — Sized and Bounded Pools

- **MUST** bound the pool and time the acquisition; an unbounded pool or a new-resource-per-request pattern is BLOCK-severity.
- **SHOULD** carry a one-line bound rationale (a Little's Law estimate, a downstream limit, or a load-test result). *Any value is acceptable with that rationale.*

```java
// WRONG — unbounded thread pool: one OS thread per task, exhausts memory under burst
ExecutorService pool = Executors.newCachedThreadPool();

// RIGHT — fixed pool with a bounded queue and an explicit rejection policy
// Bound rationale (Little's Law): ~50ms latency * 200 req/s ≈ 10 concurrent; size 16 with headroom
ExecutorService pool = new ThreadPoolExecutor(
    16, 16, 0L, TimeUnit.MILLISECONDS,
    new ArrayBlockingQueue<>(100),                 // bounded queue
    new ThreadPoolExecutor.CallerRunsPolicy());    // back-pressure on saturation
```

## Self-Review Checklist

| # | Check | Required | Rule |
|---|---|---|---|
| 1 | Every acquired resource released on every exit path (return, throw, cancel) | Required | Q12 / Q12.1 |
| 2 | Release uses a scope-bound construct, not a happy-path manual close | Required | Q12.1 |
| 3 | Cleanup is idempotent — double close/dispose/cancel is a safe no-op | Required | Q12 / Q12.2 |
| 4 | Every external/async operation has an explicit, documented timeout | Required | Q12 / Q12.3 |
| 5 | Timeout values carry a one-line rationale | Required | Q12.3 |
| 6 | Every cache/queue/buffer/accumulator declares max size | Required | Q12 / Q12.4 |
| 7 | Every cache/queue/buffer declares eviction/rejection policy + behavior at bound | Required | Q12 / Q12.4 |
| 8 | No unbounded growth on an unbounded input domain | Required | Q12 / Q12.4 |
| 9 | Every retry has a finite ceiling | Required | Q12 / Q12.5 |
| 10 | Retries use exponential backoff with jitter, or document why another strategy is used | Required | Q12.5 |
| 11 | Retries gated by a retriable-error predicate, in a dedicated wrapper | Required | Q12 / Q12.5 |
| 12 | Every spawned task/coroutine/thread tied to a lifecycle or cancellation token | Required | Q12 / Q12.6 |
| 13 | Cancellation propagates to children; no orphaned tasks | Required | Q12.6 |
| 14 | CPU-bound/polling loops check cancellation cooperatively | Required | Q12.6 |
| 15 | Fast producers apply back-pressure; no unbounded buffer between producer and consumer | Required | Q12 / Q12.7 |
| 16 | Connection/thread pools and semaphores are explicitly sized and bounded | Required | Q12 / Q12.8 |
| 17 | Pool bounds carry a one-line rationale; acquisition has a timeout | Required | Q12.8 |
