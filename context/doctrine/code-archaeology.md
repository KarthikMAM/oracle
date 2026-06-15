# Doctrine: Code Archaeology

Pre-modification investigation for `oracle-engineer`. Understanding *why* code exists in its current form is what prevents a change from silently breaking an implicit contract. This is read and applied by the engineer before any edit; the orchestrator's GATE rejects a write that skipped it.

## Procedure

For each file to be modified (skip only for greenfield where no siblings exist):

1. **Blame.** `git blame` — identify the top contributors. Authorship reveals who holds the style authority and will likely review the change.
2. **History.** `git log` on the file — the last several commits show the active direction of evolution. A change that fights that direction is a change that gets pushed back.
3. **Siblings.** Read ≥3 sibling files in the same module to infer conventions: naming, import style, the error-handling idiom in use, the test framework and fixture style.
4. **Blast radius.** Trace callers to depth 2 with Grep. Every caller is a potential regression site and must be considered in the change design.
5. **Record.** Capture an ARCHAEOLOGY block — primary author, recent commits, inferred conventions, caller count — before the pre-write read attestation.

## Rules

- Archaeology completes before the pre-write attestation.
- Conventions discovered in the surrounding code **override** the generic coding standards — team patterns win.
- If git history shows an approach was tried and reverted, do not re-propose it without new evidence that the conditions changed.
- Callers found at depth 2 that depend on current behavior MUST shape the change design, not be discovered after the fact.
