# Observability

Production code you cannot observe is production code you cannot debug; the only evidence of a 3 a.m. failure is the telemetry the code emitted before it broke. Correctness rules — never leaking a secret, never silently swallowing a failure — are MUST/MUST NOT and carry no escape hatch. Judgment-call rules (which seams deserve a metric, how a message is shaped, whether an entry point needs an inbound trace id) are SHOULD/SHOULD NOT, or MUST with an explicit documented escape hatch.

## Anchors

- **Charity Majors — Observability Engineering (2022)** — you debug unknown-unknowns from wide, structured events, not from grepping prose
- **Google SRE Book (Beyer et al., 2016)** — the four golden signals: latency, traffic, errors, saturation
- **Dapper (Sigelman et al., 2010)** — distributed traces require a correlation id propagated across every hop
- **Twelve-Factor App, Factor XI — Logs (Wiggins, 2011)** — logs are an event stream; the app writes structured events to stdout and never manages routing
- **Brendan Gregg — Systems Performance (2020)** — measure outcomes at boundaries; you cannot fix what you do not emit a metric for

## Q9.1 — Structured Logging Only

- **MUST** · every log call carries a stable severity level and a constant message, and attaches variable data as a key/value context map (an event with no variable data — `logger.info("server.started")` — needs no map).
- **SHOULD** · the message is a constant; variable data lives in the context map, never interpolated into the message. *Escape hatch: a log site you do not control — a third-party/framework logger whose signature only accepts a formatted string, or a generated-code emission point — MAY emit a non-structured line; document the constraint inline (e.g. `// framework logger: string-only signature, structure unavailable`) and keep variable data out of the message where the API allows.*

```kotlin
// WRONG — interpolated message; every line is a unique string, ungroupable
logger.info("Fetched user $userId in $elapsedMs ms from $region")

// RIGHT — static message + structured context
logger.info("user.fetch.completed", mapOf(
    "userId" to userId,
    "elapsedMs" to elapsedMs,
    "region" to region,
))
```

## Q9.2 — Banned Logging Primitives

- **SHOULD NOT** · ad-hoc output primitives appear in committed code: `print()`, `println`, `console.log` / `console.error` / `console.warn`, `Log.d` / `Log.v` / `Log.i` / `Log.w` / `Log.e`, `NSLog()`, `os_log` as a print substitute, `fmt.Println` for diagnostics, `dbg!` / `eprintln!`.
- **SHOULD** · every diagnostic routes through the project's structured logger (per Q9.1).
- **SHOULD** · intentional CLI product output — text the tool exists to print to a user — routes through a dedicated presenter/writer, not scattered `print` calls, so it stays separable from logging. *(This is the sanctioned exception: product output, not diagnostics.)*

Keeping secrets out of these primitives is not optional — that is governed by Q9.3, which stays MUST NOT regardless of the output mechanism.

```typescript
// WRONG — console.* is not a logger
console.log("connected to db")
console.error(err)

// RIGHT
logger.info("db.connection.established", { host, poolSize })
logger.error("db.query.failed", { operation: "loadOrders", errorType: err.name })
```

## Q9.3 — Never Log Secrets or PII

This is the same redaction contract the error-handling boundary enforces on error surfaces (see Error Handling Q7.9); observability and error handling share one contract.

- **MUST NOT** · log context contains secrets (passwords, tokens, API keys, session cookies, private keys, connection strings) or raw PII (email, phone, full name, government id, payment card numbers, precise location).
- **MUST** · secrets are omitted entirely — not masked, not truncated.
- **MUST** · operationally-required PII is replaced by a stable opaque reference (hashed/tokenized id), never the raw value.
- **MUST NOT** · whole-object logging (`logger.info("req", { request })`) at a trust boundary; log an explicit allowlist of fields.

```java
// WRONG — logs the raw token and the whole request object
log.info("auth.attempt", Map.of("token", bearerToken, "request", request));

// RIGHT — opaque reference, explicit allowlisted fields, secret omitted
log.info("auth.attempt", Map.of(
    "userRef", hashOf(userId),     // stable opaque id, not the email
    "tokenPresent", bearerToken != null,
    "scopes", request.getScopes()  // explicit field, not the whole object
));
```

## Q9.4 — Boundary Context and Error Paths

This is the same context the error-handling boundary emits — observability and error handling share one contract (see Error Handling Q7.4).

- **MUST** · every trust boundary (network, IPC, database, deserialization, user input, file I/O) and every error path logs at minimum: operation name and, on failure, the error type/code and the correlation/trace id (success-path correlation id is the SHOULD below).
- **MUST** · every `catch`/failure branch that does not rethrow emits one structured log with operation, correlation id, and error type. *Escape hatch: in code with no request context (a CLI, a one-shot migration), the correlation id MAY be omitted — paralleling Error Handling Q7.4; operation name and error type stay mandatory.*
- **MUST** · a failure is logged exactly once, at the boundary that handles it — not at every frame as it propagates.
- **SHOULD** · success paths on a boundary also carry the correlation id so a single request's full lifecycle is reconstructable from one trace-id filter.

```kotlin
// WRONG — error swallowed; nothing emitted, incident is invisible
when (val r = safeRun { repository.fetchUser(id) }) {
    is SafeResult.Success -> render(r.value)
    is SafeResult.Failure -> renderError("Could not load user") // silent!
}

// RIGHT — log once at the handling boundary: operation, correlation id, error type
when (val r = safeRun { repository.fetchUser(id) }) {
    is SafeResult.Success -> render(r.value)
    is SafeResult.Failure -> {
        logger.error("user.fetch.failed", mapOf(
            "operation" to "fetchUser",
            "traceId" to ctx.traceId,
            "errorType" to r.error::class.simpleName,
        ))
        renderError("Could not load user")
    }
}
```

## Q9.5 — A Metric for Every Boundary Outcome

Logs tell you what happened in one request; metrics tell you the rate and trend across all of them. These are the golden signals applied per boundary.

- **SHOULD** · every boundary operation emits a success/failure count (outcome-tagged) and a latency measurement. *Escape hatch: a trivial in-process seam with no I/O, or a hot leaf call where metric emission would dominate the measured cost, MAY be left uninstrumented with a documented inline rationale (e.g. `// hot leaf: per-element, metric emission would outweigh the work; covered by the enclosing batch metric`) provided the enclosing boundary still carries the outcome metric.*
- **SHOULD** · failure counts are emitted on every error path, tagged so success and failure are distinguishable.
- **SHOULD** · latency is measured around the boundary call (the I/O, not in-process bookkeeping) and recorded as a timer/histogram, not a single gauge.
- **MUST** · when a boundary is instrumented, the counter fires on both success and failure paths.
- **MUST NOT** · unbounded values (user id, request id, raw URL, free-text error message) are used as metric dimensions; dimensions are low-cardinality enumerable values (operation, outcome, error class, region). *Escape hatch: a higher-cardinality dimension MAY be used with a documented inline rationale stating the bounded domain and debugging need (e.g. `// tenantId: bounded to ~200 enterprise tenants, needed for per-tenant SLO`); put high-cardinality identifiers in logs/traces (Q9.4) instead.*

```typescript
// WRONG — counts only success, no latency, no failure dimension; userId as a dimension
async function loadOrders(id: string): Promise<Order[]> {
  const orders = await db.query(/* ... */)
  metrics.increment("orders.loaded")
  return orders
}

// RIGHT — outcome-tagged counter + latency on both paths, low-cardinality dimensions only;
// the failure is propagated as a typed SafeResult, never swallowed into an empty array
// (Error Handling Q7.7/Q7.13: no empty-as-complete)
async function loadOrders(id: string, ctx: Ctx): Promise<SafeResult<Order[]>> {
  const start = performance.now()
  const result = await safeRunAsync(() => db.query(/* ... */))

  // latency on every path, regardless of outcome
  metrics.timing("orders.load.latencyMs", performance.now() - start, {
    operation: "loadOrders",
  })

  if (result.success) {
    metrics.increment("orders.load", { operation: "loadOrders", outcome: "success" })
    return { success: true, value: result.value }
  }

  metrics.increment("orders.load", {
    operation: "loadOrders",
    outcome: "failure",
    errorType: result.error.name,   // bounded set of error classes
  })
  return { success: false, error: result.error }   // propagate, do not return []
}
// high-cardinality identifiers belong in the log, not the metric dimension:
logger.info("orders.load", { operation: "loadOrders", userRef: hashUser(id) })
```

## Q9.6 — Propagate the Correlation Id and Log at the Right Level

### Correlation id propagation

- **MUST** · a correlation/trace id is created at the entry boundary if the inbound request lacks one, and the inbound id is reused if present.
- **MUST** · the id crosses every async hop (callback, promise chain, coroutine, queue message, thread handoff) and every outbound service call (propagated as a header).
- **MUST** · outbound requests forward the id in the agreed propagation header so downstream services join the same trace. *Escape hatch: an entry point with no inbound id and no downstream hop — a fire-and-forget local job, a CLI, or a one-shot migration with no request context — MAY operate without a propagated id (paralleling Q9.4 and Error Handling Q7.4); document the intent inline. The instant such code makes an outbound service call or async handoff, propagation becomes mandatory again.*

### Log levels

- **SHOULD** · `ERROR` is reserved for unexpected conditions needing human attention.
- **SHOULD NOT** · expected, handled conditions (validation rejections, cache misses, optimistic-lock retries, user-not-found) are logged at `ERROR`.
- **SHOULD NOT** · `DEBUG`/`TRACE` is emitted unconditionally in hot paths (per-frame, per-element, tight loops); gate it behind a level check or sampling.
- *Reference: `WARN` is for recoverable anomalies worth noticing; `INFO` is for significant lifecycle events; `DEBUG` is for developer diagnostics off by default in production.*

```kotlin
// WRONG — id dropped across the coroutine hop; user-not-found logged at ERROR
scope.launch {
    val data = service.fetch()                           // new context, no traceId
    logger.error("user.notFound", mapOf("id" to id))     // ERROR for an expected case
}

// RIGHT — id carried across the hop; expected condition logged at INFO
scope.launch(MDCContext(mapOf("traceId" to ctx.traceId))) {
    val data = service.fetch(headers = mapOf("X-Trace-Id" to ctx.traceId))
    logger.info("user.notFound", mapOf("operation" to "fetchUser", "traceId" to ctx.traceId))
}
```

## Self-Review Checklist

| # | Check | Required | Rule |
|---|---|---|---|
| 1 | Every log call has a level + message + key/value context; message SHOULD be a constant (uncontrolled third-party/generated log sites documented inline) | Required | Q9.1 |
| 2 | Variable data is in the context map, not interpolated into the message | Required | Q9.1 |
| 3 | No `print` / `println` / `console.*` / `Log.*` / `NSLog` / `dbg!` in committed code (SHOULD NOT) | Required | Q9.2 |
| 4 | All diagnostics route through the structured logger (SHOULD) | Required | Q9.2 |
| 5 | Intentional CLI product output is separated from logging via a dedicated presenter/writer, not scattered `print` calls (SHOULD) | Required | Q9.2 |
| 6 | No secrets in logs (omitted entirely, not masked or truncated) (see Error Handling Q7.9) | Required | Q9.3 |
| 7 | PII replaced by a stable opaque reference; no whole-object logging at boundaries | Required | Q9.3 |
| 8 | Every trust boundary and error path logs operation + correlation id + error type (id MAY be omitted only where there is no request context, per Q7.4) | Required | Q9.4 |
| 9 | Each failure logged exactly once at the handling boundary (no swallowed errors, no per-frame duplicates) | Required | Q9.4 |
| 10 | Success paths on a boundary also carry the correlation id so the full request lifecycle is reconstructable (SHOULD) | Required | Q9.4 |
| 11 | Every boundary emits success/failure count + latency (SHOULD); when instrumented, the counter MUST fire on both paths (trivial/hot-leaf seams left uninstrumented carry a documented rationale) | Required | Q9.5 |
| 12 | Metric dimensions are low-cardinality; high-cardinality use has documented rationale | Required | Q9.5 |
| 13 | Correlation/trace id created at entry, reused if inbound, propagated across every async hop and service call (no-inbound-id, no-downstream entry points documented inline) | Required | Q9.6 |
| 14 | `ERROR` reserved for unexpected conditions; expected conditions not logged at `ERROR` (SHOULD / SHOULD NOT) | Required | Q9.6 |
| 15 | No unconditional `DEBUG`/`TRACE` in hot paths (SHOULD NOT) | Required | Q9.6 |
