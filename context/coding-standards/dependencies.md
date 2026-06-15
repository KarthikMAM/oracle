# Dependencies & Supply Chain

Every dependency you add is code you ship but did not write, cannot fully audit, and must trust at runtime — its bugs, its breaking changes, and its compromise become yours. Supply-chain violations are BLOCK-severity for the safety reviewer lens.

## Anchors

- **event-stream / left-pad** — a single transitive dependency, taken over or unpublished, breaks or backdoors thousands of downstream builds
- **Reproducible Builds** — the same source plus the same pinned inputs must always produce the same artifact; floating versions destroy this
- **Saltzer & Schroeder (1975)** — Economy of Mechanism: the less code in the trusted base, the smaller the attack surface
- **Rich Hickey — Spec-ulation (2016)** — dependencies are a relationship over time; breaking changes are the supplier's defect, pinning is the consumer's defense
- **Russ Cox — Surviving Software Dependencies (2019)** — a dependency is a standing decision to trust someone else's code; evaluate it like hiring
- **The Update Framework (TUF)** — integrity and authenticity of artifacts must be verifiable independent of the registry

## Q1.1 — Justify Every Dependency

- **SHOULD** answer all of these in the change description before adding a dependency: what problem it solves that the platform/stdlib and existing dependencies cannot; transitive weight (`npm ls`, `./gradlew dependencies`, `pipdeptree`, `swift package show-dependencies`, `cargo tree`, `go mod graph`); attack surface (build time, runtime, or both — network, filesystem, process environment); cost of removal (drives Q1.4).
- **SHOULD NOT** add a dependency for a single trivial function (re-implementable correctly in under ~20 lines with no platform-specific edge cases) — inline it with a test instead. *Escape hatch: if the trivial logic carries genuine correctness edge cases the team has been burned by (locale-aware casing, time-zone-correct date math, Unicode-correct widths), a vetted dependency MAY be preferred — record the specific footgun in the change description.*

```jsonc
// WRONG — a whole package and its transitive tree for one helper
"dependencies": { "is-even": "1.0.0" } // pulls is-odd -> is-number

// RIGHT — inline the trivial logic, owned and tested by us
// src/util/number.ts
export const isEven = (n: number): boolean => n % 2 === 0
```

## Q1.2 — Prefer the Platform and Standard Library

- **SHOULD** reach for the platform/stdlib before any third-party package for HTTP clients, JSON, UUIDs, base64, hashing, date/time math, collections, string formatting, and async primitives.
- **SHOULD NOT** add a third-party package that merely re-wraps a stdlib API for ergonomics unless it removes a documented, repeated correctness footgun — document the footgun, otherwise inline a thin local helper.

| Need | Platform answer (no dependency) |
|---|---|
| HTTP | `URLSession` (Swift), `HttpClient` (Java/Kotlin), `fetch` (TS), `urllib` (Python — if a richer third-party client is justified, `httpx`), `net/http` (Go) |
| JSON | `Codable` (Swift), `JSON` (TS), `json` (Python), `encoding/json` (Go); the JVM has no batteries-included stdlib JSON — `kotlinx.serialization` / Jackson are *sanctioned vetted dependencies*, not a no-dependency answer |
| UUID | `UUID()` (Swift), `java.util.UUID`, `crypto.randomUUID()` (TS), `uuid` stdlib (Python); Go has no stdlib UUID — generate RFC 4122 bytes from `crypto/rand`, or accept a vetted single-purpose dependency (`github.com/google/uuid`) |
| Hash/crypto | platform crypto only — see Security Q10 |

```python
# WRONG — third-party package for what the stdlib already does
padded = left_pad.left_pad(s, 8, "0")
# RIGHT — stdlib
padded = s.rjust(8, "0")
```

## Q1.3 — Pin Exact Versions, Lockfile, Integrity Hash

- **MUST**, in an application's or deployable's manifest, pin every direct dependency to an exact version; range operators (`^`, `~`, `>=`, `*`, `latest`, `main`, an unpinned git ref, `+` in Gradle dynamic versions) **MUST NOT** appear in that manifest.
- **MUST**, in a published library/SDK package's manifest, express minimal compatible version ranges rather than exact pins, and rely on the consuming application's committed lockfile to pin the exact resolved tree — exact-pinning a library's *direct* deps forces diamond-dependency conflicts (npm peer deps, Cargo/Maven version unification, the pip resolver) on every downstream consumer. The Rich Hickey anchor applies here: pinning is the consumer's defense, so it belongs in the consumer's lockfile, not the library's manifest. The lockfile, integrity-hash, and frozen-install MUSTs below still apply in full to the application that consumes the library.
- **MUST** commit a lockfile that is the source of truth for resolution (`package-lock.json`/`yarn.lock`/`pnpm-lock.yaml`, `Package.resolved`, `gradle.lockfile`, `poetry.lock`/`pip` hashes, `Cargo.lock`, `go.sum`).
- **MUST** carry a per-artifact integrity hash in the lockfile (SRI/`sha512`, `go.sum` checksum verified by `go mod verify`, `--require-hashes`); installs **MUST** verify against it and fail closed on mismatch.
- **MUST** install in CI in frozen/locked mode (`npm ci`, `pnpm install --frozen-lockfile`, `poetry install` against a committed `poetry.lock` optionally gated by `poetry check --lock`, `pip install --require-hashes`, `cargo build --locked`, `go build -mod=readonly`) so an out-of-date or tampered lockfile fails the build. (`go build -mod=readonly` — the default since Go 1.16 — fails when `go.mod`/`go.sum` would need to change; `go mod verify` does NOT, so it belongs only to the integrity-hash bullet above, not here.) *The only sanctioned deviation is a fully documented Q1.8 exception block.*

```jsonc
// WRONG — floating ranges; resolve to a different tree on every install
"dependencies": { "lodash": "^4.17.0", "internal-sdk": "latest", "patched-fork": "github:acme/lib#main" }

// RIGHT — exact pins; registry deps carry SRI integrity hashes in the lockfile,
// while the git fork is pinned to an immutable commit SHA, which is its integrity control
"dependencies": { "lodash": "4.17.21", "internal-sdk": "2.4.0", "patched-fork": "github:acme/lib#a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0" }
```

## Q1.4 — Wrap Third-Party APIs Behind a Thin Internal Interface

- **SHOULD** access every third-party library called from more than one module through a thin internal interface (port/adapter), not the library's types directly.
- **SHOULD NOT** let the library's concrete types leak across the seam — translate to domain types at the adapter (composes with Error-Handling Q7 and Type-Design Q4.2).
- **SHOULD** expose only the subset of the library the project actually uses, shrinking the surface a future swap must reproduce. *Escape hatch: a small, stable, single-call-site, hard-to-abstract dependency (a logging facade, a DI annotation, the platform test framework) MAY be used directly without a wrapper — record the decision (single call site, or a deliberately pervasive framework) in the change description. Pervasive frameworks chosen as architectural foundations are exempt.*

```typescript
// WRONG — library type and call bleed through the whole codebase
import dayjs from "dayjs"
function dueDate(o: Order): dayjs.Dayjs { return dayjs(o.createdAt).add(30, "day") }
// ...50 other files import dayjs directly; swapping it touches all of them

// RIGHT — thin internal port; dayjs lives behind one adapter
// clock/Clock.ts  (the seam the codebase depends on)
export interface Clock { addDays(at: Instant, days: number): Instant }

// clock/DayjsClock.ts  (the only file that imports dayjs)
import dayjs from "dayjs"
export class DayjsClock implements Clock {
  addDays(at: Instant, days: number): Instant {
    return toInstant(dayjs(at.epochMs).add(days, "day"))
  }
}
```

## Q1.5 — Vet License, Maintenance Health, and Known CVEs

- **MUST** confirm the license is on the project's allowed list and compatible with how the project ships; copyleft/viral or non-OSI licenses **MUST NOT** be introduced without explicit owner sign-off recorded in the change description. *The recorded sign-off is the documented escape hatch.*
- **MUST NOT** add a dependency whose exact pinned version has a known unpatched critical/high CVE (`npm audit`, `osv-scanner`, `pip-audit`, `cargo audit`, `govulncheck`) — no escape hatch.
- **SHOULD NOT** adopt an abandoned or single-publish package for production-critical paths; check maintenance health (recent activity, more than one maintainer where feasible, non-abandoned issue cadence, a real version history). *Escape hatch: where no maintained alternative exists, record the risk and the compensating control (vendoring, a Q1.4 seam) in the change description.*
- **MUST** run automated vulnerability scanning in CI so newly-disclosed CVEs in already-pinned dependencies are surfaced and triaged — no escape hatch.

```bash
# RIGHT — vet the exact pinned version before it lands; fail the build on critical/high
osv-scanner --lockfile=package-lock.json
npm audit --audit-level=high
pip-audit --strict
cargo audit --deny warnings
govulncheck ./...
```

## Q1.6 — No Post-Install Scripts From Unvetted Packages

- **MUST** run CI and developer installs with lifecycle scripts disabled by default (`npm ci --ignore-scripts`, `--ignore-scripts` in `.npmrc`, pip wheels over sdists, Gradle without unreviewed init scripts).
- **MUST** review and explicitly allowlist (with reason recorded) any dependency that requires a post-install/`prepare`/`preinstall`/`postinstall` script; its script **MUST** be read before allowlisting.
- **MUST NOT** execute arbitrary code from an unreviewed dependency at install time (restates and composes with Security Q10's supply-chain rule).

```ini
# RIGHT (plain npm) — .npmrc disables lifecycle scripts by default; CI installs with the same flag
# .npmrc
ignore-scripts=true
```

```jsonc
// RIGHT (pnpm) — do NOT set a global ignore-scripts (it would also block these);
// rely on pnpm's default of not building deps unless allowlisted.
// package.json
"pnpm": { "onlyBuiltDependencies": ["esbuild", "node-sass"] }  // each reviewed; reason recorded
```

## Q1.7 — Minimize Count, Surface, and Transitive Tree

- **SHOULD** prefer one well-maintained dependency over several micro-packages, and remove a dependency when its last call site is deleted; dead dependencies **SHOULD NOT** linger in the manifest. *Escape hatch: a dependency kept deliberately during a mid-flight migration, or one re-introduced shortly (a known upcoming feature), MAY remain — note the reason and the removal trigger in the change description.*
- **SHOULD** review the transitive tree when adding a direct dependency — a small direct dependency that drags in a large or risky subtree is evaluated on the whole tree it introduces. *Escape hatch: for a tiny, single-call-site dependency whose tree is trivially small, a brief note in the change description that the tree was inspected and found minimal suffices.*
- **SHOULD** prefer dependencies with zero or few transitive dependencies for equivalent functionality.
- **SHOULD NOT** let duplicate dependencies that do the same job (two HTTP clients, two JSON libraries, two date libraries) coexist — consolidate on one. *Escape hatch: a temporary duplicate during a deliberate library migration MAY coexist provided the change description names the migration, the surviving library, and the tracking ticket that removes the other.*

```bash
# RIGHT — inspect the FULL tree a new direct dependency introduces before committing
npm ls --all newpkg
./gradlew dependencies --configuration runtimeClasspath
pip install --dry-run newpkg && pipdeptree -p newpkg
cargo tree -p newpkg
go mod graph | grep newpkg
swift package show-dependencies
```

## Q1.8 — Documented Exception Block

- **MUST** carry an exception block adjacent to where a non-conforming or deviating dependency is declared (unpinned, forked, no integrity hash, or a Q1.7 minimization deviation), containing all five of: which rule is excepted (the Q1.x id); why the conforming path is not possible right now; scope (exactly which dependency/version/fork); mitigation (the compensating control — vendored + checksummed copy, pinned commit SHA, internal mirror); expiry (a linked tracking ticket and a date by which it must be resolved).
- **MUST** treat an exception missing any of the five fields as a violation.

```jsonc
// RIGHT — forked dependency with a full, time-boxed exception block
//
// DEPENDENCY EXCEPTION — Q1.3 (exact-pin) + Q1.5 (maintenance)
// Why:        Upstream `acme/widget` has an unmerged fix for the auth bypass
//             in CVE-2026-1111; no tagged release contains it yet.
// Scope:      Our fork acme-fork/widget, pinned to commit SHA below ONLY.
// Mitigation: Pinned to immutable commit SHA (not a branch); vendored copy
//             checksummed in lockfile; fork diff reviewed line-by-line.
// Expiry:     Remove fork when upstream 3.2.0 ships. https://issues.example.com/PROJ-5678 — due 2026-09-01
"widget": "github:acme-fork/widget#9f8e7d6c5b4a39281706f5e4d3c2b1a09f8e7d6c"
```

## Self-Review Checklist

| # | Check | Required | Rule |
|---|---|---|---|
| 1 | New dependency justified in change description (problem, transitive weight, attack surface, removal cost) | Required | Q1.1 |
| 2 | No dependency added for a trivial (<~20-line) function — inlined and tested instead, or the documented edge-case footgun recorded | Required | Q1.1 |
| 3 | Platform/stdlib preferred over a third-party package for HTTP/JSON/UUID/hash/dates/collections | Required | Q1.2 |
| 4 | Every direct dependency pinned to an exact version (no `^`/`~`/`>=`/`*`/`latest`/floating git ref) | Required | Q1.3 |
| 5 | Lockfile committed with per-artifact integrity hashes; CI installs frozen/locked and fails closed | Required | Q1.3 |
| 6 | Multi-call-site third-party libraries accessed through a thin internal seam; concrete types do not leak | Required | Q1.4 |
| 7 | Direct-use exceptions (single call site / pervasive framework) recorded in the change description | Required | Q1.4 |
| 8 | License on allowed list and compatible; viral/non-OSI licenses signed off | Required | Q1.5 |
| 9 | Pinned version has no unaddressed critical/high CVE; CI runs continuous vulnerability scanning | Required | Q1.5 |
| 10 | Maintenance health checked (activity, maintainers, version history) before adoption | Required | Q1.5 |
| 11 | Installs run with lifecycle scripts disabled by default | Required | Q1.6 |
| 12 | Any required post-install script reviewed and explicitly allowlisted with reason | Required | Q1.6 |
| 13 | Transitive tree reviewed when adding a direct dependency; prefer few/zero transitive deps | Required | Q1.7 |
| 14 | No dead dependencies and no duplicate dependencies doing the same job (or a migration duplicate documented per the escape hatch) | Required | Q1.7 |
| 15 | Any non-conforming or deviating dependency carries a 5-field exception block (rule, why, scope, mitigation, expiry) | Required | Q1.8 |
