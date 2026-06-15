# Review Target: design

The artifact under review is a **design or spec** — `design.md`, `requirements.md`, or `tasks.md` produced by oracle-architect. This is prose and architecture, not code, so the attack patterns are different from a code diff. Review only from this perspective: do not hunt force-unwraps or formatting — hunt for a design that is unsound, incomplete, ambiguous, or untraceable.

## Attack patterns

Apply every pattern. For each, record `clean` (with the specific evidence why), `FINDING` (with the exact location — section/requirement id — impact, and a concrete fix), or `N/A` (with a one-line justification).

1. **logical-soundness** — trace the design's own control and data flow. Find the step that does not follow, the case the approach cannot handle, the assumption that, if false, collapses it. A design that only works on the happy path is unsound.
2. **requirement-traceability** — every requirement (`R<n>`) maps to a concrete design element, and every design element traces back to a requirement. Flag orphans in both directions: a requirement no design element satisfies, and a design element no requirement asks for (scope creep).
3. **completeness** — every failure mode, consumer, and non-functional obligation the goal implies is addressed: error paths, rollback, security, observability, accessibility, cost, migration, backward compatibility. A design silent on how it fails in production is incomplete.
4. **ambiguity** — any statement a competent engineer could implement two materially different ways is a defect. A spec is an instruction, not a suggestion; flag every place the reader must guess.
5. **testability** — every requirement is stated so a test can prove it (EARS form: Ubiquitous / Event-driven / State-driven / Unwanted). Flag any requirement that cannot be verified as written.
6. **alternative-coverage** — at least two real alternatives were considered, each with a concrete rejection reason (not "we preferred A"). A thin or missing rejection means the chosen approach has not earned its place.
7. **blast-radius** — the design names every downstream caller, consumer, wire format, and platform it touches, and how compatibility is preserved. Flag an unanalyzed blast radius.
8. **dag-validity** (`tasks.md`) — the task DAG is acyclic; every touched file appears in exactly one task; files within a wave are disjoint; package dependency order is consistent with `depends_on`. Flag a cycle, an orphan file, or an intra-wave file collision.
9. **intent-fidelity** — the design solves the problem the user actually stated, honors stated constraints, and surfaces every deviation for a decision. (This is the `context/review/intent-fidelity.md` lens, which applies across targets — apply it here.)

## What does NOT apply

Code-diff patterns — force-unwrap hunting, complexity counting, layout/formatting, line-level type safety — are `N/A` for a design (there is no code yet). Record them N/A rather than forcing them.

## Procedure and output

Follow the standard reviewer procedure (`agents/oracle-reviewer.md`) and the cross-cutting rules in `context/review/_shared.md` — confidence scoring, steel-man on every BLOCK, SHOULD-accumulation, finding format, exhaustiveness attestation. Cite the specific requirement id, design section, or task id for every finding. Route by root cause: a missing requirement or wrong assumption routes to `intake`; an unsound or incomplete approach routes to `design`.
