# Review Target: research

The artifact under review is **research output** — a `findings.md` from oracle-researcher or the synthesized `intake.md`. This is evidence, not code and not architecture. Review only from this perspective: do not assess implementation or design quality — assess whether the findings are *true, sufficient, and honestly bounded*. Bad research silently poisons every decision downstream of it.

## Attack patterns

Apply every pattern. For each, record `clean` (with the specific evidence why), `FINDING` (with the exact claim and its location, the gap, and a concrete fix), or `N/A` (with a one-line justification).

1. **evidence-sufficiency** — every load-bearing claim is backed by at least the `thresholds.research_cross_reference_floor` independent sources. A single-source claim that the design will rest on, presented above low confidence, is a finding.
2. **falsification-presence** — every finding records a real counter-evidence search (what was looked for to disprove it), not a token gesture. A finding with no falsification attempt is incomplete — the most confident-looking claim with no disconfirming search is the most dangerous.
3. **citation-validity** — every cited path or URL resolves and was fetched in this session (present in the evidence index for external sources). Flag any claim sourced from memory, any dead citation, any "as is well known."
4. **currency** — sources are current for a fast-moving topic; superseded docs, deprecated APIs, and stale threads are flagged as such rather than cited as live truth.
5. **confidence-calibration** — the stated confidence matches the evidence: high confidence requires cross-confirmed independent sources; single-source or inferred claims are capped at low/medium. Flag overconfidence.
6. **gap-honesty** — what could not be answered is stated plainly, not papered over. A research memo that presents partial findings as complete is a finding; an honest "could not determine X, here is why" is not.
7. **scope-fidelity** — the findings answer the angle that was actually asked, without drifting into a different question or smuggling in the researcher's preferred conclusion. (Composes with `context/review/intent-fidelity.md`.)

## What does NOT apply

Code-diff and design patterns — force-unwraps, complexity, DAG validity, formatting — are `N/A` here; there is no code or design in a research memo. Record them N/A.

## Procedure and output

Follow the standard reviewer procedure (`agents/oracle-reviewer.md`) and the cross-cutting rules in `context/review/_shared.md` — confidence scoring, steel-man on every BLOCK, finding format, exhaustiveness attestation. Cite the specific claim and source for every finding. Route by root cause: insufficient or unfalsified evidence routes back to `intake` (more or sharper research); a finding that exposes a wrong framing of the question also routes to `intake`.
