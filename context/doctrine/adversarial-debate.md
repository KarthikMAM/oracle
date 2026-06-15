# Doctrine: Adversarial Debate

How the orchestrator pits a producer against a dedicated adversary until the work can't be broken. This is the depth mechanism — distinct from the breadth of lens audits. Read and applied by oracle-dev (drives the debate), oracle-critic (attacks designs), and oracle-tester (attacks code).

## Two modes of scrutiny

Oracle scrutinizes work two ways, and both must pass:

- **Audit (breadth)** — `oracle-reviewer` walks a lens's checklist asking "does this violate a known-bad pattern?" Many lenses, applied in parallel, cover the whole surface.
- **Debate (depth)** — a dedicated adversary picks a goal and tries to *make the artifact fail*. `oracle-critic` attacks the design; `oracle-tester` attacks the code. One adversary, driven deep, until it lands a hit or runs out of attacks.

An artifact is done only when the auditors find nothing **and** the adversary cannot land a hit.

## The debate loop

The orchestrator runs this dialectic at the DESIGN gate (architect vs. critic) and the CODE gate (engineer vs. tester):

1. **Propose.** The producer returns its artifact (design or diff).
2. **Attack.** The orchestrator spawns the adversary on that artifact, forwarding any prior rebuttals so it attacks what changed and confirms prior hits are truly closed.
3. **Judge.** The orchestrator reads the attack and decides — independently, as the harshest reviewer would. A landed hit is one with a concrete, reproducible scenario; a vague worry is not a hit. The orchestrator is the judge, not the adversary: it can rule an attack invalid (and record why), and it can rule a SURVIVED verdict premature if the adversary skipped an attack class.
4. **Route.** A landed hit goes back to the producer (DESIGN or CODE) with the scenario. The producer fixes the root cause and re-proposes. Repeat from step 2.
5. **Settle.** The debate ends when the adversary returns SURVIVED with a full breadth-and-depth attestation and the orchestrator concurs — or when a loop budget is hit, which is an ESCALATE, not a silent pass.

## Rules

- The adversary attacks; it never fixes. It names the break and proves it; the producer owns the remedy. This keeps the critic independent of the author.
- Every landed hit is concrete and reproducible — a failing scenario for a design, a runnable reproduction for code. "This might be slow" is not a hit; "this loses the second write when the process dies between line 40 and 44, here's the sequence" is.
- The adversary steel-mans before it strikes: state the artifact's best defense, then defeat it or concede. This prevents cheap-shot findings.
- The debate is bounded by the same loop budgets and oscillation guard as any fix loop (`context/quality-gates.md`). A debate that won't converge is a design-level disagreement — ESCALATE with both sides, don't loop forever.
- **Debate depth scales to measured blast radius (breadth never does).** For a change with a demonstrably small, low-risk blast radius — single file, no public surface, no wire format, no concurrency, no security-sensitive path — the orchestrator MAY reduce debate rounds toward `thresholds.min_debate_rounds`, recording the gating decision in `judge-verdict.json` (the verdict-auditor checks it). This gates on the *measured* surface the diff touches, never on a gut read of "simple." The lens audit and all quality gates still run in full — only the expensive adversarial debate flexes. Anything above the trivial bar runs full debate depth. See `context/principles.md` → "Full pipeline, every task."
- Audit and debate are complementary, not redundant. Run the lens audit and the debate both; neither substitutes for the other. The breadth-and-depth mandate in `context/principles.md` requires both.

## Grounded evidence, or it's advisory

Acting on *ungrounded* feedback is how a correcting system makes correct work worse: with no external signal, intrinsic correction does not reliably improve and often degrades (Huang et al., "LLMs Cannot Self-Correct Reasoning Yet", ICLR 2024), and a debate can talk a system *out of* a correct answer. So:

- **Grounded findings are actioned; ungrounded findings are advisory.** Before the orchestrator routes a finding back to a producer, the finding MUST be backed by grounded evidence — a failing test, a runnable reproduction, a cited standard rule, or a cited spec/requirement line. A finding with no executable or cited grounding is recorded and surfaced to the producer as *advisory* (worth considering), not auto-actioned as a required fix. It is never silently dropped — it is just not allowed to force a change on argument alone.
- **Do not regress on argument alone.** Any change that would reverse an earlier-passing state — green tests, a prior SURVIVED verdict, an approved design element — MUST be justified by *new grounded evidence*, not by an adversary's rhetoric. If the only thing that changed is that someone argued more persuasively, the earlier passing state stands. This is the orchestrator's regression-check (audited by `target=verdict`).
- **Weight executable verdicts above no-oracle verdicts.** A tester reproduction and the build/test/coverage signal are asymmetrically verifiable — they have an executable oracle — so they carry more weight than a design critique that has none (debate's leverage comes from verification being easier than generation — Kenton 2024). Where no executable oracle exists, debate still adds value: a non-expert judge weighing two adversaries reaches the truth more often than reading either alone (Khan 2024, "Debating with More Persuasive LLMs Leads to More Truthful Answers") — which is exactly why an ungrounded critic stays in the loop as a *check*, even though it cannot by itself force a regression. A tester BLOCK with a reproduction always halts. An ungrounded critic BLOCK that the producer disputes goes to the regression/advisory rule above and, if still contested, ESCALATEs to the user with both sides rather than forcing a redesign.
