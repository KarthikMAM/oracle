# clean-code

Layout, formatting, vertical spacing, signature wrapping, comment placement, diff hygiene. Drives the formatting contract Self-Review Checklist.

## Why

Formatting rules exist because consistency reduces cognitive load. A reader scanning a codebase shouldn't have to mentally normalize five different signature-wrap styles. These are not aesthetic preferences — they're shared conventions that compound across a codebase.

## Canonical Contract

The reviewer reads the binding formatting contract from the project (or plugin default):
- `.oracle/coding-standards/formatting.md` (or `context/coding-standards/formatting.md`)

Findings MUST cite a specific rule ID (`F<N>`) and quote the verbatim rule text.

## Attack Patterns

1. **signature-layout-compliance**: Single-line when ≤3 params, no complex types, fits limit (F1). Multi-line otherwise with each param on own line, trailing comma where supported.
2. **method-chain-layout**: Chains exceeding 2 calls broken across lines (one call per line). Chains of ≤2 calls stay on one line if they fit (F2).
3. **single-expression-function-form**: Multi-line single-expression functions use block body with explicit `return`, not `=` expression spanning multiple lines (F3).
4. **argument-layout**: Sole-statement-in-block calls with 3+ arguments use multi-line argument formatting. Unused parameters replaced with `_` (F4).
5. **vertical-spacing-production**: Production code body separated into Guard/Setup/Transform/Side-effect/Return blocks with one blank line between, none inside (F5).
6. **vertical-spacing-test**: Test body in Arrange/Act/Assert (or Given-When-Then) blocks. Exactly one Act per test. No assertions before Act. (F5).
7. **brace-spacing**: No blank line after `{` or before `}` (F5).
8. **comment-placement**: Inline comments on line above, not end-of-line (F6). No `// region` / `// endregion` markers.
9. **diff-hygiene**: No reformat-only churn outside touched hunks. No mixed reformatting + behavioral changes in one CR (F7).
10. **trailing-comma**: Multi-line param/argument lists use trailing commas where the language supports (Kotlin, TypeScript) for cleaner diffs.

## Severity

- **BLOCK**: ≥3 violations from same rule (systemic neglect). Mixed reformatting + behavioral changes in one CR.
- **SHOULD**: Individual formatting violation that obscures structure.
- **NIT**: Minor layout improvement, optional polish.

## Finding Format

Templates:
- "Signature at {file:line} wrapped to multi-line with 2 params and no complex types — F1 mandates single-line. Reformat."
- "Method chain at {file:line} has 4 calls on one line — F2 mandates one call per line when >2 calls. Break."
- "Function `{name}` at {file:line} body lacks blank line between Setup and Transform blocks — F5 mandates one blank line per logical block."
- "Comment at {file:line} placed end-of-line — F6 mandates comments on line above. Move."

## Critical Rules

- Findings MUST cite a specific F-number rule.
- Layout violations are SHOULD by default; only escalate to BLOCK on systemic neglect (≥3 from same rule).
- "Looks ugly" is not a finding — cite the rule.
