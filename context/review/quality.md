# quality

Is the code *substance* sound? Drives the substance Self-Review Checklists: function shape, types and classes, error handling, documentation, naming, testing, dependency hygiene. Layout/formatting is the **clean-code** lens's contract, not this one — this lens does not duplicate it. Every finding MUST cite a specific rule ID.

## Why

Code quality rules exist because every violation has caused a production defect at least once. Force-unwraps crash. Functions >20 lines accumulate bugs. Missing doc comments force reverse-engineering. These are defect-prevention guardrails proven by experience.

## Canonical Contracts

The reviewer loads only the standards **this lens validates**, intersected with what the diff touches — never the whole corpus. The `quality` lens validates code *substance*, so it loads the substance standards: `function-shape.md`, `naming.md`, `error-handling.md`, `documentation.md`, `testing.md`, `type-design.md` and `class-design.md` (when the diff defines types/classes), and `dependencies.md` (when it adds one). It does **not** load `formatting.md` — layout is the `clean-code` lens's contract, not this one — nor any conditional standard whose surface the diff doesn't touch (a UI tweak's reviewer skips the migration rules). Read `context/coding-standards/_index.md` once to map a standard to its trigger, then load only the subset your patterns below actually cite against this diff. Loading a standard you won't cite is wasted context.

Findings MUST cite a rule ID (`Q<N>.<M>` or `F<N>` or section header) and quote the verbatim rule text, and treat **every** Self-Review Checklist row as Required (the checklists have no optional items). No inventing rules outside these contracts.

## Attack Patterns

1. **documentation-completeness**: Every new/modified function and type has a complete doc comment (Q2.1). Every parameter documented. No TODO without ticket. No commented-out blocks >3 lines (Q2.2).
2. **function-shape-limits**: size, nesting, complexity, ternary, and `else`-block limits per function-shape Q3.1–Q3.2 — cite the standard's current thresholds and strengths (MUST vs SHOULD + escape hatch), do not restate numbers here (they drift).
3. **type-safety-audit**: No force unwraps/casts/tries (Q3.1). No `any` in TypeScript (Q4.3). No non-null assertion `!`. No compiler/linter silencing annotations.
4. **type-centralization**: No inline object/union/intersection/tuple types in signatures (Q4.2). No `Pair`/`Triple` returns. All types in module's centralized types file.
5. **naming-compliance**: Functions = verb phrases. Types = noun phrases. Booleans = `is`/`has`/`should`/`can`/`did`/`will` (Q3.3). No banned names.
6. **error-handling-pattern**: No ad-hoc `runCatching` in business logic (Q7). No discarded `SafeResult`. Boundary functions own try/catch; business logic throws.
7. **logging-compliance**: No banned logging primitives (`print()`, `console.log()`, `Log.d()`, `NSLog()`). Structured logging with level + static message + K/V context.
8. **test-structure**: AAA / Given-When-Then. One Act per test. No assertions before Act. Test naming `test_<method>_<scenario>_<expected>`.
9. **dependency-hygiene**: New dependencies justified, version pinned, wrapped behind internal interface. No `latest`, `*`, `^0.x`.
10. **comment-provenance-detection**: Every code comment, doc string, test name in the diff MUST be free of internal pipeline provenance:
   - No requirement IDs in process context ("per R5", "implements R3")
   - No task hex slugs with pipeline anchors ("per design 8f13fbd8")
   - No process-narration phrases ("user requirement", "per the conversation")
   - No AI-edit artifacts ("AI-edited", "agent-generated", "machine-generated")
   - No oracle agent names (oracle-engineer, oracle-reviewer, oracle-dev, etc.)
   - No pipeline paths (`$HOME/.oracle/`)
   Severity: BLOCK for any match.
11. **simplicity-and-clarity** (cites `context/principles.md` → "Simplicity — the cross-cutting value"): flag code that chooses cleverness over the obvious solution, hidden/implicit control flow or side effects, a premature or wrong abstraction where a little duplication would read clearer, and a new dependency or framework pulled in for something small and stable the code could own. Applied in the language's own idiom — the bar is "would a tired maintainer understand this on first read at 3am," not adherence to any one language's syntax. Severity: SHOULD by default (clarity is judgment), escalating to BLOCK when the obscurity could hide a defect (a clever expression whose edge case is unclear, an abstraction that buries an error path).

## Severity

- **BLOCK**: Force-unwrap, `any`, hardcoded secret, banned logging primitive, silencing annotation, comment-provenance leakage.
- **SHOULD**: function over the size/complexity limits in function-shape Q3.1–Q3.2 (cite the standard's current numbers), banned name, missing visibility, anemic class / Feature Envy / inlined infrastructure mechanism (class-design).
- **NIT**: Missing doc comment on private helper, minor naming inconsistency.

## Finding Format

Templates:
- "Force-unwrap at {file:line} — Q3.1 prohibits force-unwraps. Fix the type system so it proves the value is present."
- "Function `{name}` at {file:line} is {N} lines — exceeds the Q3.1 size limit. Extract a helper."
- "Inline type `{ key: Type }` at {file:line} — Q4.2 mandates type centralization. Extract to types file."
- "Comment `{text}` at {file:line} references pipeline process — comment-provenance-detection. BLOCK."

## Critical Rules

- Findings without a rule ID MUST NOT be emitted.
- Rules NOT in the canonical contracts MUST NOT be invented here.
- Quote the verbatim rule text alongside the ID.
