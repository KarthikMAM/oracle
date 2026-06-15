# Review Target: verdict

The artifact under review is **the orchestrator's own judgment** — a `judge-verdict.json` in which oracle-dev recorded why it ruled a debate SURVIVED, accepted a producer's PASS, or made a consequential advance/route/escalate decision. This is the one place nothing else audits: oracle-dev judges every gate, and a single unaudited judge is a documented blind spot (failure attribution tops out ~53% even with structured logs — Who&When; "incorrect verification" is a named MAST failure mode; debate's value collapses when the judge has no independent check — Kenton). Review only from this perspective: do not re-review the underlying code or design — review whether the *judgment about it* was sound and unbiased.

## What you are given

- `judge-verdict.json` — the orchestrator's recorded rationale: which attacks/findings landed, which it ruled invalid and why, what state it advanced from/to, and the decision it made.
- The underlying artifacts it judged (the design or diff, the adversary/reviewer findings, the prior verdict it may have reversed).

## Attack patterns

Apply every pattern. For each, record `clean` (with the specific evidence why), `FINDING` (quoting the judge-verdict line at issue, the bias/gap, and the correction), or `N/A`.

1. **unevidenced-concurrence** — every "the design survived" / "advance" conclusion rests on cited evidence (a ruled-invalid attack names *why* it's invalid; a SURVIVED cites the attacks driven to ground). Flag any verdict that concurs by assertion — "looks sound", "no further concerns" — without pointing at what was checked.
2. **dismissed-finding-audit** — for every adversary/reviewer finding the orchestrator ruled invalid or downgraded, verify the stated reason actually defeats it. A real, grounded finding waved away is the most expensive judge error.
3. **regression-check** — if this verdict reverses an earlier-passing state (green tests, a prior SURVIVED, an approved design element), verify the reversal is justified by *new grounded evidence*, not by an adversary's argument alone (debate can overturn a correct answer — iMAD). Flag an unjustified reversal.
4. **self-preference** — the orchestrator authored the commit/CR text and synthesized intake itself. Where it judges its *own* output, check it held that output to the same bar as a sub-agent's (LLM judges favor their own generations — Panickssery). Flag a softer bar on self-authored artifacts.
5. **premature-consensus** — flag a SURVIVED reached before the adversary pursued every attack class its module defines, or a debate closed after one shallow round when the artifact's risk warranted more.
6. **position / recency bias** — flag a verdict that tracks the last or longest argument rather than the strongest one; the orchestrator should weigh by grounding, not order or verbosity.
7. **executable-vs-rhetorical weighting** — verify the orchestrator weighted grounded/executable verdicts (a tester reproduction, failing tests, a cited standard) above no-oracle rhetorical arguments (an ungrounded design critique). Flag a redesign forced by argument over a passing executable signal.
8. **lens-selection audit (code-gate verdicts only)** — the four mandatory lenses always run, but the *conditional* lenses (performance, compatibility, platform-compliance) are spawned on the orchestrator's read of the diff's surface. A conditional lens that is silently never spawned produces no finding, so nothing else can catch its omission — this is the one place a whole class of defect can go unexamined. Read the `judge-verdict.json` lens-set record against the actual diff: if the change touches a public/exported/wire/serialized/removed-symbol surface, `compatibility` MUST have run; a hot path or new unbounded loop/allocation, `performance`; a platform/UI surface, `platform-compliance`. Flag any conditional lens the surface implicated but the record shows skipped — or any skip whose stated evidence does not actually hold against the diff. A missing lens-set record on a code-gate verdict is itself a BLOCK (the selection was never put on the record to audit).

## Severity & routing

Severity per `context/review/_shared.md` (BLOCK / SHOULD / NIT, steel-man on every BLOCK). A BLOCK here routes **back to the orchestrator's decision itself** — it re-judges with the bias corrected, it does not route to a producer. A pattern of the same judge error across verdicts is systemic and ESCALATEs to the user with both sides.

## What does NOT apply

Code-diff, design, and research patterns — this target audits the *judgment*, not the artifact judged. Record them N/A. (The lens-selection audit, pattern 8, is the one place you inspect the diff's *surface* — but only to check whether the recorded lens-selection decision was sound, never to re-review the code itself.)

## Procedure and output

Follow the standard reviewer procedure (`agents/oracle-reviewer.md`) and the cross-cutting rules in `context/review/_shared.md`. Cite the specific `judge-verdict.json` line for every finding.
