# Review Target: code

The artifact under review is a **code diff** — modified source produced by oracle-engineer. Review it through the eight code lenses; apply the patterns of your assigned lens in full and review only from that perspective.

## Lenses

Each lens is a file of concrete attack patterns. The orchestrator spawns one reviewer per lens (parallel), so you are assigned exactly one — read it and apply every pattern in it.

| Lens | File | Hunts |
|---|---|---|
| correctness | `context/review/correctness.md` | wrong output, missing edge cases, requirement gaps |
| safety | `context/review/safety.md` | concurrency, untrusted input, resource & crypto hazards |
| performance | `context/review/performance.md` | hot-path overhead, allocations, N+1, unbounded growth |
| quality | `context/review/quality.md` | the coding-standards substance contracts (every rule id) |
| compatibility | `context/review/compatibility.md` | API back-compat, caller breakage, wire format, platform matrix |
| intent-fidelity | `context/review/intent-fidelity.md` | does the change solve the problem the user actually stated |
| platform-compliance | `context/review/platform-compliance.md` | SDK availability, lifecycle, framework idioms, a11y-as-contract |
| clean-code | `context/review/clean-code.md` | layout, formatting, diff hygiene (the formatting contract) |

## Mandatory lenses

Regardless of what the change looks like, these four are **mandatory** — skipping one is a fan-out gap (quality-gates G15): **correctness, safety, intent-fidelity, quality**. The other four apply whenever the change plausibly touches their domain — `performance` for hot paths, `compatibility` for public-surface or wire-format changes, `platform-compliance` for platform/UI code, `clean-code` always for layout. A reviewer that finds its lens genuinely inapplicable records why; it does not silently drop it.

## Standards coverage

The `quality` and `clean-code` lenses enforce the coding-standards corpus, and each loads **only the standards its own lens validates**, intersected with what the diff touches (per `context/coding-standards/_index.md`) — never the whole corpus. The `quality` lens owns *substance* (function-shape, naming, error-handling, documentation, testing, type-design, class-design, dependencies) and cites `Q<n>.<m>` / `C<n>`; the `clean-code` lens owns *layout* (`formatting.md` only) and cites `F<n>`. The two do not overlap — formatting findings come only from clean-code, substance findings only from quality. Every Self-Review Checklist row in a standard a lens loads is **Required** — there are no optional items. Each reviewer quotes the verbatim rule text alongside the id. Loading a standard the lens won't cite, or whose surface the diff doesn't touch, is wasted context.

## Procedure and output

Follow the standard reviewer procedure (`agents/oracle-reviewer.md`) and the cross-cutting rules in `context/review/_shared.md` — confidence scoring, steel-man on every BLOCK, SHOULD-accumulation, rule citation, exhaustiveness attestation, finding format. Route every finding by root cause (intake / design / code / ship).
