# Shared Reviewer Rules

This file documents rules that apply to **every** lens in `context/review/`. Each lens file inherits these rules — they are not redundantly repeated in every lens, but the reviewer agent applies them universally regardless of lens.

## Review target — review the artifact you were given, not a generic one

Reviewing a code diff, a design, and a research memo are **different activities** with different failure modes. The orchestrator gives the reviewer a `target`, and the reviewer dynamically loads the matching module — `context/review/targets/{code,design,research}.md` — and reviews **only** from that perspective. Applying one artifact type's patterns to another (a code-diff hunt against a prose design, or vice versa) is a category error and a fan-out gap (quality-gates G15).

| Target | Module | Patterns |
|---|---|---|
| `code` | `context/review/targets/code.md` | the eight lens files; mandatory: correctness, safety, intent-fidelity, quality |
| `design` | `context/review/targets/design.md` | soundness, traceability, completeness, ambiguity, testability, alternatives, DAG-validity |
| `research` | `context/review/targets/research.md` | evidence-sufficiency, falsification, citation-validity, currency, calibration, gap-honesty |
| `verdict` | `context/review/targets/verdict.md` | audits the orchestrator's own judgment: unevidenced-concurrence, dismissed-finding, regression, self-preference, premature-consensus, position/recency bias, executable-vs-rhetorical weighting, lens-selection |

## Load only what your lens validates

A reviewer is spawned for **one lens** and loads only the context that lens needs to cite — never the whole standards corpus. Your lens file (`context/review/<lens>.md`) names its own binding contracts; load those and stop. Most lenses (`correctness`, `safety`, `performance`, `compatibility`, `intent-fidelity`, `platform-compliance`) work primarily from their own attack-pattern list and the diff, pulling in a specific standard only when a pattern cites one directly. The two standards-enforcing lenses are scoped and disjoint: `quality` loads the *substance* standards (function-shape, naming, error-handling, documentation, testing, type-design, class-design, dependencies); `clean-code` loads only `formatting.md`. Within whatever a lens loads, narrow further by the diff — skip a conditional standard whose surface the change doesn't touch (`context/coding-standards/_index.md` maps each to its trigger). Reading a standard you will not cite against this diff is wasted context and blurs which lens owns a finding.

The module defines the patterns; the rules below apply to a reviewer on **any** target.

## Confidence Scoring (all lenses)

For each finding, assign a confidence score 0-100:

| Score | Meaning |
|---|---|
| 0-25 | Likely false positive or pre-existing issue |
| 26-50 | Minor nitpick not in coding standards |
| 51-75 | Valid but low-impact |
| 76-90 | Important issue requiring attention |
| 91-100 | Critical bug or explicit standards violation |

Only report findings with confidence ≥ **80**. Findings 51-79 are recorded internally for trend analysis but NOT surfaced to the fix loop.

**Exception**: BLOCK-severity findings are always reported regardless of confidence (severity overrides confidence threshold for blockers).

## Steel-Man Counter-Argument (all lenses)

For every BLOCK finding, the reviewer MUST:

1. Write the strongest counter-argument (why it might be a false positive).
2. If counter-argument wins → downgrade or remove.
3. If counter-argument loses → BLOCK stands, with documented adversarial challenge.

## SHOULD-Accumulation (all lenses)

If ≥ **3** SHOULD-severity findings cluster in the same lens → escalate verdict to BLOCK (systemic neglect).

This is mechanical — the reviewer MUST NOT rationalize it away.

## Rule Citation (all lenses)

Every finding MUST cite a rule. Citation forms:
- Coding-standards rule ID (`Q3.1`, `Q4.7`, `F5`, etc.) for `quality` and `clean-code` lenses.
- Pattern ID from the lens file (`boundary-values`, `trust-boundary-trace`, etc.) for all other lenses.
- Spec line reference (`requirements.md §R5`) for `intent-fidelity` lens.
- RFC clause (`RFC 9110 §9.2.1`) for protocol-conformance findings.

Findings without a rule citation MUST NOT be emitted.

## Exhaustiveness Attestation (all lenses)

Before declaring PASS, the reviewer MUST emit an attestation listing every attack pattern from the lens file with status:

| # | Pattern | Status | Evidence |
|---|---|---|---|
| 1 | {pattern_name} | clean / FINDING / N/A | {Why clean, or → Finding N, or why N/A} |

Plus the declaration:

> Every attack pattern in {lens} has been applied. No pattern was skipped without explicit N/A justification. No further attack avenue exists within this lens.

A verdict without exhaustiveness attestation is a BLOCK condition (silent failure).

## Finding Format (all lenses)

Findings MUST include:
- **Severity**: BLOCK | SHOULD | NIT
- **Confidence**: 0-100
- **Attack Pattern**: which pattern caught this
- **Rule Citation**: contract clause / pattern ID / spec line / RFC clause
- **Location**: file:line
- **Issue**: what's wrong, with evidence
- **Impact**: what fails and under what conditions
- **Suggested Fix**: concrete fix, not vague guidance
- **Steel-Man Counter-Argument** (BLOCK findings only): strongest argument that this is a false positive
- **Resolution** (BLOCK findings only): why the counter-argument loses or downgrades

## Concern is a SEED, Not a CAGE (all lenses)

The `concern` the orchestrator hands the reviewer seeds the investigation but does NOT limit scope. The reviewer MUST apply ALL attack patterns from the lens file regardless of what the concern says.
