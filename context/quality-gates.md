# Quality Gates — Orchestrator Rejection Registry

`oracle-dev` applies these checks to every sub-agent return *before* trusting it. A return that matches a failure pattern is rejected and the agent is re-spawned with the diagnostic prepended. Sub-agent verdicts are input, not truth — the orchestrator's own judgment is the gate.

## How to use this registry

After every spawn returns:

1. Read the artifact(s) on disk the agent claims to have produced.
2. Check the relevant gates below.
3. On a match: append a `violation` line to the manifest, re-spawn with the listed diagnostic, and prepend the diagnostic to the new spawn message.
4. If the same gate fails more than `thresholds.infra_retry_limit` times for the same target, stop re-spawning and `ESCALATE` to the user with the evidence.

## Gates

| # | Gate | Trigger | Diagnostic on re-spawn |
|---|---|---|---|
| G1 | **Phantom artifact** | The reply claims `done`/`PASS` but the cited artifact does not exist on disk. | "Artifact missing at `<path>`. Produce the file, then return its path." |
| G2 | **Empty research** | The researcher returned without invoking any tool from its assigned medium's tool list. | "Investigate using medium=`<X>` tools listed in config `research.<X>.tools`; cite ≥ `research_cross_reference_floor` independent sources." |
| G3 | **Medium confusion** | The researcher's findings cite tool calls outside its assigned medium. | "Stay within your assigned evidence medium; re-run with medium=`<X>` tools only." |
| G4 | **Evidence not persisted** | Findings cite an external URL/doc with no matching entry in `evidence/index.jsonl`. | "Persist every fetched external artifact verbatim to `evidence/` and index it before citing it." |
| G5 | **Uncited claim** | A load-bearing claim in any artifact has no `file:line` or URL backing it. | "Every load-bearing claim cites `file:line` or a URL from a tool call in this session. Fix the uncited claim: `<text>`." |
| G6 | **No falsification** | A researcher finding lacks a counter-evidence search. | "Run the mandatory falsification step: search for evidence that disproves each finding; record what you searched even when it found nothing." |
| G7 | **Missing attestation** | Any agent returned PASS without the exhaustiveness attestation. | "Emit the exhaustiveness attestation (avenues tried, zero-result queries, no-further-avenue declaration) before PASS." |
| G8 | **Pre-write skipped** | The engineer wrote source without first completing archaeology (the ARCHAEOLOGY block — primary author, recent commits, inferred conventions, caller count, per `context/doctrine/code-archaeology.md`) and the read attestation (binding standards, every modified file, its tests, its callers). | "Complete archaeology (the ARCHAEOLOGY block) and the read attestation (standards + every modified file + its tests + its callers) before any Write/Edit." |
| G9 | **Self-review skipped** | The engineer returned PASS without walking the Self-Review Checklists of the standards it touched. | "Re-read the relevant coding standards and walk every Self-Review Checklist item before verdict=PASS." |
| G10 | **Unverified green** | A green claim has no captured tool output to back it — the engineer's scoped tests/typecheck/lint, or the builder's full build/test/coverage. | "Re-run the check and cite the actual captured output (log path) for every green claim — scoped checks for the engineer, the full build/test/coverage for oracle-builder." |
| G11 | **Scope breach** | Diff touches a file outside the task's declared scope, or any `policy.off_limits_globs` path. | "Revert out-of-scope edits; modify only files in the task. Off-limits paths MUST NOT be written." |
| G12 | **Provenance leak** | Commit message, PR/CR text, code comment, or test name references Oracle, an agent, `$HOME/.oracle/`, a task slug, a requirement ID, or AI authorship. | "Remove all pipeline/AI provenance; write as the engineer. BLOCK." |
| G13 | **Editorialized spawn** | (self-check) The orchestrator injected a conclusion or dropped a constraint in a spawn message. | Re-issue the spawn with faithful, neutral context per `context/communication.md`. |
| G14 | **Forbidden git op** | A `policy.forbidden_git` operation (`push --force`, `reset --hard`, `clean -fd`, `--no-verify`, …) was attempted. | "Forbidden git operation. Use the safe alternative; never force-push, hard-reset, or skip hooks." |
| G15 | **Insufficient fan-out** | Research used fewer than `research_fanout_floor` angles; or a review used the wrong `target` module for the artifact, or skipped a mandatory pattern for that target (the modules in `context/review/targets/` — e.g. code mandates correctness/safety/intent-fidelity/quality); or covered fewer than `review_fanout_floor` lenses on a code diff. | "Decompose into ≥ `research_fanout_floor` independent angles / load the correct target module and cover every mandatory pattern; re-fan-out the missing ones." |
| G16 | **Breadth-incomplete** | The work addressed the named ask but left part of the affected surface untouched — an un-updated call site of a changed signature, an unhandled edge case, a sibling that needed the same change, a consumer of a changed contract — or the breadth attestation is missing. | "Enumerate and cover the full affected surface (every call site, consumer, edge case, sibling); emit the breadth attestation. A symptom-only fix is incomplete." |
| G17 | **Shallow fix** | The fix masks the symptom instead of the root cause — a guard at the crash site rather than the contract that admitted the bad value, a special-case for the one reported input, a silenced error, a design flaw papered over in code — or the depth attestation is missing. | "Trace the root cause and fix that, not the symptom; emit the depth attestation naming the cause you fixed." |

Gates G16 and G17 enforce the broad-and-deep mandate in `context/principles.md`. They bind every producer (researcher, architect, engineer) and every adversary (the attack must cover the whole surface and drive to root cause, not stop at the seed). The orchestrator applies them after reading the artifact — a "looks done" that fails either is rejected, not advanced.

## Loop budgets

Every loop in the pipeline has an independent, un-bypassable ceiling. The orchestrator tracks each and `ESCALATE`s when one is crossed — because the most expensive failure is a run that churns without converging. Each loop has its OWN counter; no counter is shared across two loops (a shared counter lets one loop's spend mask another's runaway).

| Budget | Counts | Ceiling | On exceed |
|---|---|---|---|
| Per-phase iterations | Re-spawns within INTAKE / DESIGN / CODE | `max_research_rounds` / `max_design_iterations` / `max_code_iterations` | ESCALATE — the phase can't reach a clean state on its own. |
| Debate rounds (per gate) | Consecutive propose→attack→judge rounds in one DESIGN or CODE debate — incremented every round regardless of whether the adversary's specific finding changed | `max_debate_rounds` | ESCALATE — an adversary returning a *new* BROKEN each round (deadlock-by-attrition) is a sign the artifact or the spec is fundamentally contested; the user decides. This is the backstop the per-`(file,lens)` oscillation guard cannot provide for in-phase debate. |
| Verdict re-audit | Times a `target=verdict` BLOCK sends a single decision back to the orchestrator to re-judge | `max_verdict_reaudits` | ESCALATE — the orchestrator and its own auditor disagree on the same call repeatedly; the user breaks the tie. |
| Backward routes | Every route of a finding to an *earlier* phase (CODE→DESIGN, DESIGN→INTAKE, and a human CR comment sending SHIP→DESIGN or SHIP→INTAKE), across the whole run | `max_backward_routes` total | ESCALATE — repeated backward routing means the problem is mis-framed, not mis-built. |
| Ship polls | Poll iterations for the CR (within one releaser spawn) | `max_ship_iterations` × `ship_poll_seconds` | ESCALATE — CI is stuck or the change can't go green. |
| Ship fix-rounds | Releaser↔engineer round-trips on the CR (a CR comment → code fix → re-push → re-poll cycle), across all spawns | `max_ship_fix_rounds` | ESCALATE — a CR that surfaces a *different* comment each amend (so the oscillation guard never trips) is a sign the change can't satisfy review; the user decides. |

Backward routes are the most dangerous: a CODE→DESIGN→CODE ping-pong where each leg presents a *different* `(file, lens)` defeats the oscillation and cycle guards while making no progress — the backward-route budget is the backstop the per-`(file,lens)` oscillation guard cannot provide.

**When DESIGN is re-entered from a later phase:** the architect regenerates `tasks.md`. The orchestrator diffs the new DAG against the prior one and re-runs only the tasks whose `files` or design section actually changed — completed, still-valid tasks are not redone. This depends on the per-task verdicts the resume model already records.

**When INTAKE is re-entered from CODE or SHIP** (a "missing info" route): it runs only the sharper angle the finding named, updates `intake.md`, and returns to the phase that routed back to it — *not* to a fresh user checkpoint (the run is already past APPROVE; re-entry is autonomous unless it surfaces a genuinely non-researchable question, which ESCALATEs).

## Oscillation

When the same `(file, lens)` pair produces a finding across `thresholds.oscillation_limit` consecutive fix iterations, the fix loop is not converging. The orchestrator stops looping and `ESCALATE`s to the user — this is usually a design-level disagreement masquerading as a code nit.

## The escalation packet

Every `ESCALATE` to the user (red baseline, repeated gate failure, oscillation, budget exceeded, cross-workspace coupling, stuck CI) presents the same structure, so the user can act after a long autonomous run without re-deriving context:

1. **Trigger** — which budget/gate/condition fired, in one line. *(Always present.)*
2. **What was attempted** — the avenues tried and why each failed, citing artifacts. *(Present when a loop/fix history exists; for a pre-flight ESCALATE with no history, state "none — failed at the gate.")*
3. **Both sides** — for a disputed finding, the steel-manned argument for and against. *(Required when the trigger is a disputed finding — oscillation, debate deadlock, verdict-reaudit, coupling; omit when there is no two-sided dispute, e.g. red baseline or stuck CI.)*
4. **The decision** — the single question the user must answer. *(Always present.)*
5. **Options** — enumerated, each with its consequence. *(Always present.)*

Parts 1, 4, and 5 are always present; parts 2 and 3 are present when applicable to the trigger (some triggers — red baseline, unresolvable build, stuck CI — have no attempt-history or two-sided dispute to show). An ESCALATE missing an *applicable* part is incomplete — the user should never have to ask "what do you need from me?"

## Cycle & depth safety

- **Depth:** only `oracle-dev` spawns; sub-agents never spawn. Spawn depth is therefore exactly 1. Any sub-agent attempting to spawn is a defect.
- **Cycle:** the same `(agent, target, input-hash)` tuple MUST NOT be spawned twice within a run. The manifest is the record; a repeat means a fix loop is stuck — `ESCALATE` instead.
