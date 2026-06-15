# correctness

Does the artifact produce its intended outcome? Spec-vs-intent fidelity, coverage, edge cases, exhaustiveness, requirement traceability, error-handling completeness.

## Why

Correctness is the most fundamental property of software — code that produces wrong output is worse than code that crashes, because wrong output propagates silently before anyone notices. This lens catches logical errors, missing edge cases, and requirement gaps that would otherwise reach production undetected.

## Attack Patterns

You MUST apply all attack patterns below. For each, record clean / FINDING / N/A with evidence.

1. **spec-vs-intent-reread**: Re-read `intake.md` then `requirements.md`. Flag anything in intake silent in requirements. Flag any requirement whose intent drifts from the original ask.
2. **design-vs-spec-fidelity**: Walk every requirement ID. Verify `design.md` addresses it with a concrete design element. Flag requirements with no traceability link.
3. **diff-vs-design-fidelity**: Compare the implementation diff against what `design.md` specifies. Flag deviations — both additions not in design and omissions design mandates.
4. **negative-trace**: For each happy path in code, ask: what is the failure path? Is it handled? Does the error propagate or get swallowed? Flag unhandled failure modes.
5. **boundary-enumeration**: List min, max, zero, one, max+1, zero-1 for every numeric/length input. Verify each is handled or documented as out-of-range.
6. **state-explosion**: Enumerate the cross-product of input states (nullable × empty × error × concurrent). Flag any unhandled cell.
7. **null-safety-after-suspension**: For every `await`/`suspend` point, identify references that could become nil/null between suspension and next access. Flag unguarded post-suspension access.
8. **exhaustiveness-check**: Every `if` MUST have an `else` or early-return guard. Every `switch`/`when`/`match` MUST be exhaustive. Every loop MUST handle zero iterations.
9. **missing-requirements-scan**: Walk `design.md` features. Flag any feature without a corresponding requirement ID. Flag any requirement with no test coverage.
10. **error-handling-completeness**: Every trust boundary (bridge, network, IPC, user input, deserialization, file I/O) MUST have error handling.

## Severity

- **BLOCK**: Concrete failure case with repro (input X → wrong output Y). Must fix.
- **SHOULD**: Handled inelegantly — works but masks root cause or surprises readers.
- **NIT**: Handled correctly but undocumented or unconventional.

## Finding Format

Templates:
- "Requirement R{N} (`{text}`) has no corresponding implementation in the diff — silent omission."
- "Input `{value}` at boundary (zero/max/empty) reaches {file:line} unguarded — produces {wrong output}."
- "After `await` at {file:line}, reference `{var}` may be nil — unguarded access at {file:line+1}."
- "Trust boundary at {file:line} ({type}) has no error handling — failure propagates as crash."

## Critical Rules

- BLOCK findings MUST include a concrete repro path (specific input → specific wrong output).
- "Could be wrong" without a demonstrated failure case is SHOULD at most.
- Spec-fidelity findings MUST cite the specific requirement ID or intake line that is violated.
- Every finding MUST reference specific file:line evidence.
