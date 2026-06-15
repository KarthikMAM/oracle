# Concurrency

Concurrency bugs are non-deterministic, survive code review, and are nearly impossible to reproduce — a race that fires once per million requests is invisible in test and catastrophic at scale. Explicit thread contracts, structured scoping, and a single documented synchronization mechanism per piece of shared state prevent entire categories of races before they reach production.

## Anchors

- **Structured Concurrency** (Sustrik, 2016; Swift SE-0304) — a task's lifetime is bounded by its lexical scope, so cancellation and cleanup are guaranteed and orphans are unrepresentable
- **Sendable** (Swift SE-0302) — values crossing an isolation boundary must be safe to share; the compiler proves it instead of the reviewer guessing
- **Java Concurrency in Practice** (Goetz et al., 2006) — every shared mutable field has a documented thread-safety policy, a happens-before relationship, and a single guarding lock; compound check-then-act is not atomic
- **The Rules of Hooks** (React, Abramov et al., 2018) — hook call order is a positional invariant; breaking it corrupts component state silently
- **Communicating Sequential Processes** (Hoare, 1978) — prefer message passing and confinement over shared mutable memory; share by communicating, do not communicate by sharing

## Q8 — General Rules

- **MUST** declare any thread/executor contract on public methods, properties, or callbacks in enforceable form: a compiler-checked annotation (`@MainActor`, `@MainThread` / `@AnyThread` / `@WorkerThread`), a type-level isolation marker, or — only where no annotation exists — a doc comment whose first line states the contract.
- **MUST NOT** dispatch results to the main/UI thread from service-layer, repository, or domain code; the caller owns the observation thread. *Escape hatch: a component whose entire contract is UI presentation may dispatch to main, provided that contract is documented per the rule above.*
- **MUST** tie every async scope to a lifecycle or cancellation token; fire-and-forget async with no owner, handle, or cancellation path **MUST NOT** be used (see Resource Management Q12.6).
- **MUST** protect shared mutable state with exactly one documented synchronization mechanism, named at the declaration site (which lock, which actor, which thread confines it).
- **MUST** perform any compound operation (check-then-act, read-modify-write, get-or-create, iterate-then-mutate) atomically under a single hold of its guarding mechanism (see Q8.6).
- **MUST NOT** use an unchecked thread-safety escape hatch (`@unchecked Sendable`, `@Suppress("...")` on a concurrency lint, `// nolint`, raw `unsafe` shared access, `nonisolated(unsafe)`) without an adjacent comment naming the external invariant that makes the access safe and why the compiler cannot see it.
- **MUST NOT** strongly capture `self` / `this` in escaping closures stored on long-lived objects (see Q8.5).
- **MUST NOT** run a blocking call (synchronous I/O, lock acquisition that can wait, `Thread.sleep`, `Future.get()` / a `.await()` that never yields, `runBlocking`) on the main/UI thread or on a thread reserved for non-blocking work (event loop, `Dispatchers.Default`) (see Q8.7). A plain `await` / `.await()` at a real suspension point *suspends* rather than *blocks* and is the prescribed pattern on the main actor — it is not a blocking call.
- **MUST** acquire locks in a single global order across the codebase, and a thread holding one lock **MUST NOT** acquire another out of that order or invoke unknown caller-supplied code (an "alien call") while holding it (see Q8.8).
- **MUST NOT** use a `volatile` / `@Volatile` field or a single atomic to coordinate more than one variable or to make a compound action atomic (see Q8.6).

## Q8.1 — Swift Concurrency

- **MUST** use `async let` or `withTaskGroup` / `withThrowingTaskGroup` for parallel work; **SHOULD** use `Task {}` only when work must outlive the enclosing scope, and then **MUST** store the handle so its owner can cancel it. *Escape hatch: a top-level event entry point with no enclosing async scope (e.g. a SwiftUI `.task` modifier, which binds lifetime to the view) is structured by the framework and needs no manual handle.*
- **MUST** give every long-running or looping `Task` a cancellation path: `try Task.checkCancellation()` or test `Task.isCancelled` at each iteration / before each expensive step.
- **MUST** conform any type crossing an actor or task boundary to `Sendable`, and **MUST** have `@Sendable` closures capture only `Sendable` values by value; `@unchecked Sendable` only with the Q8 escape-hatch comment.
- **MUST NOT** use `Task.detached {}` merely to escape `await`; permitted only when isolation inheritance is genuinely wrong, and then its handle **MUST** be stored and cancelled by the owner.
- **SHOULD** confine mutable state shared across concurrency domains to an `actor` or `@MainActor` type rather than a manual lock. *Escape hatch: a hot path measured to be dominated by actor-hop cost may use a documented lock instead, with the Q8 single-mechanism rule still in force.*

```swift
// WRONG — detached task, no handle, no cancellation: an uncancellable orphan
func start() {
    Task.detached { await self.pollForever() }   // survives deinit, mutates dead state
}

// RIGHT — structured, owned, cancellable
private var pollTask: Task<Void, Never>?
func start() {
    pollTask = Task { [weak self] in
        while !Task.isCancelled {                 // cooperative cancellation
            guard let self else { return }
            await self.pollOnce()
        }
    }
}
deinit { pollTask?.cancel() }
```

## Q8.2 — Kotlin Coroutines

- **MUST** launch every coroutine in a lifecycle-bound or injected `CoroutineScope` (`viewModelScope`, `lifecycleScope`, a component-owned scope, or one passed in); `GlobalScope.launch` / `GlobalScope.async` **MUST NOT** appear in production code.
- **SHOULD** inject a `CoroutineDispatcher` (constructor parameter defaulting to the real one) rather than hardcoding `Dispatchers.IO` / `Default` / `Main` inside functions. *Escape hatch: a `companion`-level constant naming the single dispatcher for a leaf utility is fine; the target is scattered hardcoded dispatchers through business logic.*
- **MUST** make every `suspend` function main-safe — move blocking or CPU-bound work onto an appropriate dispatcher with `withContext` internally, never block the caller's thread.
- **MUST** call `ensureActive()` (or check `isActive`, or use a cancellable suspend point) each iteration of any CPU-bound or unbounded loop inside a coroutine.
- **MUST** rethrow `CancellationException` from any `catch (e: Exception)` / `catch (e: Throwable)` inside a coroutine (or catch the specific business exception instead). This applies only at the one sanctioned raw-catch site — the `safeRun` body, which already encodes the rethrow (Error Handling Q7.1/Q7.8); business-logic coroutines call `safeRun` and branch on `SafeResult` rather than hand-rolling the catch.
- **MUST NOT** let parallel sub-tasks outlive their parent; fan them out under `coroutineScope { }` (fail-fast) or `supervisorScope { }` (isolated failures), and **SHOULD NOT** manually launch siblings into an outer scope for parallelism. *Escape hatch: a child whose lifetime is deliberately longer than the spawning function may be launched into an explicitly owned, lifecycle-bound scope, provided that ownership and its cancellation path are documented at the launch site per the Q8 async-scope rule.*
- **MUST** `await` every `async` `Deferred` on every path, or use `launch` if no result is needed.

```kotlin
// WRONG — GlobalScope, hardcoded dispatcher, swallowed cancellation, blocking in suspend
fun load() = GlobalScope.launch(Dispatchers.IO) {
    try {
        val data = blockingFetch()          // blocks; suspend contract violated upstream
        view.show(data)
    } catch (e: Exception) {                // swallows CancellationException -> leak
        log.warn("ignored", e)
    }
}

// RIGHT — injected scope + dispatcher, main-safe suspend, failure handled at the boundary.
// Failures go through the sanctioned `safeRun` util (Error Handling Q7.1), which already
// re-throws CancellationException for free — no hand-rolled raw try/catch in business logic.
class Loader(
    private val scope: CoroutineScope,
    private val io: CoroutineDispatcher = Dispatchers.IO,
) {
    fun load() = scope.launch {
        when (val result = safeRun { withContext(io) { repository.fetch() } }) {  // blocking work off main
            is SafeResult.Success -> view.show(result.value)
            is SafeResult.Failure -> view.showError()
        }
    }
}
```

## Q8.3 — React Hooks Discipline

- **MUST** call hooks at the top level of the component or custom hook, in the same order on every render — never inside conditions, loops, nested functions, `try` blocks, or after an early `return`.
- **MUST** list every reactive value (prop, state, context, or derived value) referenced inside `useEffect`, `useMemo`, `useCallback`, or `useImperativeHandle` in its dependency array; suppressing the exhaustive-deps lint **MUST NOT** hide a missing dependency. *Escape hatch: a deliberately run-once mount effect MUST carry an inline comment stating why the empty array is correct and that the omitted values are stable for the component's lifetime.*
- **MUST** return a cleanup function from every `useEffect` that creates a persistent resource (subscription, timer, listener, observer, async request, WebSocket); effects whose async result sets state after unmount **MUST** guard the post-unmount write (a `cancelled` flag or `AbortController`).
- **SHOULD** give a value passed as a hook dependency or to a memoized child a stable identity across renders when its content is unchanged (`useMemo` / `useCallback`). *Escape hatch: cheap primitives need no memoization; only memoize where a profiler or an effect-loop shows it matters.*

```tsx
// WRONG — conditional hook + missing dep + no cleanup + post-unmount setState
function Profile({ id, show }: Props) {
  if (!show) return null
  const [user, setUser] = useState<User>()          // hook after early return -> corrupts slots
  useEffect(() => {
    fetchUser(id).then(setUser)                       // 'id' missing from deps; sets state after unmount
  }, [])
  return <Name user={user} />
}

// RIGHT — unconditional hooks, exhaustive deps, cancellation-guarded cleanup
function Profile({ id, show }: Props) {
  const [user, setUser] = useState<User>()
  useEffect(() => {
    const controller = new AbortController()
    fetchUser(id, controller.signal).then((u) => {
      if (!controller.signal.aborted) setUser(u)
    })
    return () => controller.abort()                   // cleanup on unmount / id change
  }, [id])                                            // exhaustive
  if (!show) return null                              // early return AFTER all hooks
  return <Name user={user} />
}
```

## Q8.4 — TypeScript: No Floating Promises

- **MUST** `await`, return, `.then()`/`.catch()`-chain, or explicitly discard with `void promise` plus a justifying comment every `Promise`.
- **MUST NOT** place a Promise in a boolean or conditional position (`if (promise)`, `promise ? a : b`, `!promise`); a Promise is always truthy.
- **SHOULD** combine independent concurrent promises with `Promise.all` / `Promise.allSettled` rather than `await`ing sequentially in a loop. *Escape hatch: sequential `await` is correct when each iteration depends on the previous, or when bounded concurrency is required to avoid overwhelming a dependency — in which case bound it, e.g. with a concurrency limiter, per Resource Management Q12.7.*
- **MUST NOT** make a `forEach` / `map` callback `async` when the caller assumes the work completed; use `for...of` with `await`, or `Promise.all(array.map(async ...))`.

```typescript
// WRONG — floating promise, async forEach, promise in boolean position
function save(items: Item[]) {
  items.forEach(async (i) => { await repo.put(i) })   // all promises orphaned, errors lost
  if (repo.flush()) return                            // flush() returns a Promise -> always truthy
}

// RIGHT — awaited, fan-out BOUNDED (items is caller-sized; unbounded Promise.all would
// open one connection per element), boolean uses the resolved value
type SaveResult = { ok: true } | { ok: false; error: FlushFailure }
async function save(items: Item[]): Promise<SaveResult> {
  await mapWithConcurrency(items, MAX_INFLIGHT, (i) => repo.put(i))  // awaited; rejections propagate

  const ok = await repo.flush()                       // await the value, then branch
  if (!ok) return { ok: false, error: new FlushFailure() }   // expected outcome -> value, not a throw (error-handling.md Q7.6)
  return { ok: true }
}
```

> A bare `Promise.all(xs.map(...))` over an externally-sized collection is itself a defect — it issues one in-flight operation per element with no ceiling (see Resource Management Q12). Bound the fan-out with a concurrency limiter or chunked batches.

## Q8.5 — Closure Capture

- **MUST** capture `self`/`this` weakly (`[weak self]`, weak listener, or by extracting only the needed values) in an escaping closure stored on a long-lived object (subscription, observer, completion handler, callback registry), and **MUST** handle the weak reference being nil.
- **MUST NOT** let a `[weak self]` closure silently no-op on the common path; if the work must complete regardless, capture the specific dependencies by value, or promote to a strong reference (`guard let self`) only for the duration of the call.

```swift
// WRONG — strong self capture in escaping closure: retain cycle
func subscribe() {
    eventBus.subscribe { event in
        self.handle(event)   // bus retains closure retains self forever
    }
}

// RIGHT — weak self, nil handled
func subscribe() {
    eventBus.subscribe { [weak self] event in
        guard let self else { return }
        self.handle(event)
    }
}
```

## Q8.6 — Atomicity of Compound Operations

- **MUST** perform a check-then-act, read-modify-write, or get-or-create on shared state as one atomic step: a single critical section under the guarding lock, a single atomic compare-and-set / `compute` / `merge`, or a concurrent collection's atomic method (`putIfAbsent`, `computeIfAbsent`, `getOrPut` under a lock).
- **MUST NOT** use a `volatile` / `@Volatile` field or a single `Atomic*` to coordinate more than one variable or stand in for mutual exclusion; if an invariant spans two fields, both **MUST** be updated under one lock or packed into one atomically-swapped immutable value.
- **MUST** use a language-sanctioned safe-publication mechanism for lazy / double-checked initialization (`lazy`/`by lazy`, `lazy var` on a value confined to one thread, `LazyThreadSafetyMode.SYNCHRONIZED`, `dispatch_once`/`static let`, `sync.Once`, `OnceLock`) rather than a hand-rolled `if (instance == null)` double-check.

```kotlin
// WRONG — check-then-act race: two threads both see absent and both create
fun getOrCreate(key: String): Session {
    if (!sessions.containsKey(key)) {       // thread A and B both pass here
        sessions[key] = Session(key)         // one Session silently overwrites the other
    }
    return sessions.getValue(key)
}

// RIGHT — single atomic operation on a concurrent map
private val sessions = ConcurrentHashMap<String, Session>()
fun getOrCreate(key: String): Session =
    sessions.computeIfAbsent(key) { Session(it) }   // create-once, atomic
```

```java
// WRONG — volatile guards visibility, not atomicity: ++ is read-modify-write
private volatile int count;
void incr() { count++; }                    // lost updates under contention

// RIGHT — atomic read-modify-write
private final AtomicInteger count = new AtomicInteger();
void incr() { count.incrementAndGet(); }
```

## Q8.7 — Keep Blocking Work Off Reserved Threads

- **MUST NOT** run synchronous I/O, a lock acquisition that can wait, `Thread.sleep`, `runBlocking`, `Future.get()`/`.await()` without a yield, or any unbounded compute on the main/UI thread; offload it (`withContext(Dispatchers.IO)`, a background `Task`/`DispatchQueue`, `worker_threads`, a goroutine) and observe the result back on the UI thread (see Q8 — the offloader, not the service, dispatches back).
- **MUST NOT** run blocking I/O on a dispatcher/executor reserved for CPU-bound non-blocking work (`Dispatchers.Default`, a Node event loop, a fixed compute pool); use the dedicated I/O dispatcher/pool, sized per Resource Management Q12.8.
- **MUST** signal blocking work in the function's contract (a `suspend` boundary that internally `withContext`-es, a `@WorkerThread` annotation, or a doc comment), so callers cannot invoke it from a reserved thread by accident.

```kotlin
// WRONG — blocking disk read on the main thread freezes the UI
fun onClick() {
    val text = File(path).readText()    // synchronous I/O on Main -> ANR
    label.text = text
}

// RIGHT — read off-main, update UI on-main
fun onClick() = viewModelScope.launch {
    val text = withContext(Dispatchers.IO) { File(path).readText() }
    label.text = text                   // back on Main
}
```

```typescript
// WRONG — CPU-bound parse blocks the Node event loop; all requests stall
app.get("/report", (_req, res) => {
  const result = parseHugeCsvSync(readFileSync(path))   // blocks the loop for seconds
  res.json(result)
})

// RIGHT — offload CPU work to a worker thread, keep the loop free
app.get("/report", async (_req, res) => {
  const result = await runInWorker("parseCsv", path)    // event loop stays responsive
  res.json(result)
})
```

## Q8.8 — Deadlock Avoidance: Lock Ordering and No Alien Calls

- **MUST** acquire multiple locks in one consistent global order at all sites (e.g. by a stable id, address, or documented rank).
- **MUST NOT** make an "alien call" — invoke a caller-supplied callback, override, listener, or any method that could re-enter the subsystem or acquire another lock — while holding a lock; capture what you need under the lock, release it, then make the call.
- **MUST** keep a critical section to the minimum that preserves the invariant — never around I/O, allocation-heavy work, or unrelated computation; a blocking lock acquisition that can wait **SHOULD** use a bounded `tryLock`-with-timeout on contended paths. *Escape hatch: an uncontended, leaf-level lock with no nested acquisition may use a plain `lock`/`synchronized`; the timeout requirement targets paths where deadlock or unbounded wait is plausible.*

```kotlin
// WRONG — alien call under lock + inconsistent ordering = deadlock + reentrancy
fun transfer(from: Account, to: Account, amount: Long) {
    synchronized(from) {
        synchronized(to) {                 // path A locks from->to
            from.debit(amount)
            to.credit(amount)
            listener.onTransfer(from, to)  // alien call while holding both locks
        }
    }
}
// elsewhere transfer(b, a, ...) locks to->from -> opposite order -> deadlock

// RIGHT — global lock order by id; no alien call inside the critical section
fun transfer(from: Account, to: Account, amount: Long) {
    val (first, second) =
        if (from.id < to.id) Pair(from, to) else Pair(to, from)   // stable global order
    synchronized(first) {
        synchronized(second) {
            from.debit(amount)
            to.credit(amount)
        }
    }
    listener.onTransfer(from, to)          // alien call AFTER locks released
}
```

## Self-Review Checklist

| # | Check | Required | Rule |
|---|---|---|---|
| 1 | Thread-restricted public methods/callbacks declare an enforceable threading contract | Required | Q8 |
| 2 | Service/repository/domain code does not dispatch results to the main/UI thread | Required | Q8 |
| 3 | No fire-and-forget async without an owner, handle, or cancellation path | Required | Q8 / Q8.1 |
| 4 | Shared mutable state has exactly one synchronization mechanism, named at the declaration | Required | Q8 |
| 5 | Compound operations (check-then-act, read-modify-write, get-or-create) are atomic | Required | Q8 / Q8.6 |
| 6 | No unchecked thread-safety escape hatch without an invariant-naming comment | Required | Q8 |
| 7 | No strong self/this captures in escaping closures on long-lived objects | Required | Q8 / Q8.5 |
| 8 | No blocking call on the main/UI thread or a non-blocking-reserved dispatcher | Required | Q8 / Q8.7 |
| 9 | Locks acquired in one global order; no alien call while holding a lock | Required | Q8 / Q8.8 |
| 10 | `volatile`/single atomic not used to coordinate multiple variables or for mutual exclusion | Required | Q8 / Q8.6 |
| 11 | Swift: structured concurrency by default; `Task {}` only when lifetime must escape, handle stored | Required | Q8.1 |
| 12 | Swift: long-running/looping Tasks check cancellation | Required | Q8.1 |
| 13 | Swift: `Sendable` on types crossing actor/task boundaries; `@unchecked` justified | Required | Q8.1 |
| 14 | Swift: no `Task.detached` without stored, cancellable handle | Required | Q8.1 |
| 15 | Swift: shared mutable state confined to an `actor`/`@MainActor` rather than a manual lock where practical | Required | Q8.1 |
| 16 | Kotlin: no `GlobalScope`; coroutines in lifecycle-bound or injected scope | Required | Q8.2 |
| 17 | Kotlin: dispatchers injected (SHOULD), not hardcoded in business logic | Required | Q8.2 |
| 18 | Kotlin: every `suspend` function is main-safe (blocking work via `withContext`) | Required | Q8.2 |
| 19 | Kotlin: CPU-bound loops call `ensureActive()` | Required | Q8.2 |
| 20 | Kotlin: `CancellationException` is never swallowed by a broad catch | Required | Q8.2 |
| 21 | Kotlin: parallel work uses `coroutineScope`/`supervisorScope`; every `async` is awaited | Required | Q8.2 |
| 22 | React: hooks called unconditionally at top level, same order every render | Required | Q8.3 |
| 23 | React: exhaustive dependency arrays; no lint suppression hiding a missing dep | Required | Q8.3 |
| 24 | React: useEffect returns cleanup; async effects guard against post-unmount setState | Required | Q8.3 |
| 25 | React: hook-dependency / memoized-child values have stable identity (`useMemo`/`useCallback`) so effects do not re-fire | Required | Q8.3 |
| 26 | TypeScript: no floating Promises; `void` discards carry a justifying comment | Required | Q8.4 |
| 27 | TypeScript: no Promise in a boolean/conditional position; no `async forEach` | Required | Q8.4 |
| 28 | Lazy/double-checked initialization uses a sanctioned safe-publication mechanism | Required | Q8.6 |
| 29 | Blocking work signalled in the function's contract; offloaded then observed on UI thread | Required | Q8.7 |
| 30 | Critical sections are minimal; contended waiting locks use bounded `tryLock`/timeout | Required | Q8.8 |
