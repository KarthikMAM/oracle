# performance

Does this regress, and is the cost reasoned about or accidental? Hot-path overhead, allocations, N+1, layout thrashing, async wait chains, cache hits, memory pressure.

## Why

Performance regressions are invisible until they accumulate past a threshold — then they manifest as user complaints and "the app feels slow" tickets impossible to attribute to a single change.

## Measurement Default Inversion

For any change touching a hot path (per-frame/per-event/per-request), the reviewer MUST NOT accept "looks fine" — the producer MUST have measured or the reviewer flags it.

If the producer has NOT provided measurement data:
- Hot path: flag as SHOULD with suggested measurement approach
- Critical hot path (per-frame or per-event): flag as BLOCK

For cold paths (once at init, per-session, per-user-action): principles-based reasoning is sufficient.

## Attack Patterns

1. **hot-path-identification**: Classify each changed location by call frequency. Trace callers upward to framework lifecycle entry points. Flag changes in hot paths without measurement evidence.
2. **allocation-in-render-path**: Object creation in per-frame/per-event paths. Autoboxing in tight loops. String concatenation in N-iteration.
3. **n-plus-one-detection**: Loops containing DB queries, RPC calls, or network requests. Flag where a single batched call would replace N sequential.
4. **layout-thrashing**: In UI code, identify interleaved reads and writes to layout-affecting properties. Flag read-write-read-write patterns.
5. **sequential-await-chain**: Identify sequential `await`/`suspend` calls where operations are independent.
6. **loop-invariant-computation**: Identify computations inside loops whose result does not change per-iteration.
7. **unbounded-collection-growth**: Every cache, queue, and buffer MUST have max size, eviction policy, or TTL.
8. **sync-io-on-main-thread**: Identify synchronous I/O on main/UI thread. Flag as BLOCK — always.
9. **missing-memoization**: Identify expensive computations repeated with identical inputs in same lifecycle.
10. **measurement-evidence-check**: For every hot-path change, verify the producer included measurement data.

## Verdict Logic

| Condition | Verdict |
|---|---|
| Sync I/O on main thread | BLOCK |
| Unbounded collection on unbounded input domain | BLOCK |
| Hot-path change without measurement (per-frame/per-event) | BLOCK |
| Hot-path change without measurement (non-critical but frequent) | SHOULD |
| Sequential awaits where parallel feasible | SHOULD |
| Loop-invariant hoisting opportunity | SHOULD |
| Cold-path allocation concern | NIT |

## Finding Format

Principles-based findings MUST include:
- File:line of the concern
- Call frequency classification
- Reasoning chain
- Whether measurement data exists or is missing

Templates:
- "Sync file read at {file:line} on the main thread (per {file:line} runs during viewDidLoad). BLOCK. Move to background queue."
- "Unbounded cache at {file:line} keyed by user-supplied URL — no TTL, no eviction. BLOCK; add eviction policy."
- "Loop at {file:line} rebuilds Regex per iteration over a list typically N=200. SHOULD; hoist outside the loop."

## Critical Rules

- Principles-based findings MUST cite file:line evidence and a reasoning chain.
- "Looks slow" without reasoning is not a finding.
- Measurement-by-default inversion: ABSENCE of measurement on hot paths is itself a finding.
- Cold-path concerns MUST NOT be elevated to BLOCK without evidence of user-visible impact.
