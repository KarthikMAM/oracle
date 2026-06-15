# Coding Standards — Relevance Index

Seventeen standards govern the code Oracle writes. Loading all seventeen on every task is wasteful — a TypeScript UI tweak does not need the database-migration rules, and a one-file refactor does not need the API-versioning rules. Load the **core** always; load the rest **only when the change touches their surface**. Every standard still binds when it applies — this index decides *applicability*, not *authority*. When in doubt whether one applies, load it.

## Core — load on every code change

These are language-agnostic and apply to essentially any change:

- `function-shape.md` — size, nesting, complexity, parameters
- `naming.md` — the naming taxonomy
- `error-handling.md` — boundary error handling (`safeRun`/`Result`)
- `documentation.md` — doc comments and inline comments
- `testing.md` — test structure, coverage, mutation
- `formatting.md` — layout, vertical spacing, diff hygiene
- `security.md` — secrets, injection, trust boundaries (cheap to apply, expensive to miss)

## Conditional — load when the change touches the trigger

| Standard | Load when the change… |
|---|---|
| `type-design.md` | defines or changes types, in a typed language (TS/Kotlin/Swift/Java/Rust/Go) |
| `class-design.md` | defines or changes a class — responsibility, sizing, cohesion, inheritance/composition, encapsulation, DI |
| `concurrency.md` | touches async, threads, coroutines, locks, or shared mutable state |
| `performance.md` | touches a hot path (per-frame / per-request / per-event), a loop over large N, or allocation-sensitive code |
| `api-design.md` | changes a public API, an exported surface, or a wire/serialization format |
| `accessibility.md` | touches UI (a `.tsx`/`.jsx` component, a Swift/Compose view, any user-facing control) |
| `observability.md` | adds or changes logging, metrics, tracing, or a service/boundary |
| `resource-management.md` | opens a handle/connection, adds a cache/queue/buffer/pool, or spawns a task with a lifecycle |
| `data-integrity.md` | persists data, touches a transaction, a migration, or a mutating endpoint |
| `dependencies.md` | adds, removes, or version-changes a third-party dependency |

## How to use this

The engineer reads the core set plus every conditional standard whose trigger the task plausibly hits — based on the files, languages, and surfaces in the task slice and what archaeology surfaced. Reviewers on the `quality`/`clean-code` lenses do the same when auditing a diff: cite rules from the core plus whichever conditional standards the diff actually implicates. Erring toward loading one more standard is cheap; missing an applicable one is not.
