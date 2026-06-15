# compatibility

Does this break consumers, downstream packages, platforms, or contracts? API back-compat, caller verification, platform quirks, i18n, accessibility-as-contract.

## Why

Compatibility defects are insidious: they compile and test successfully in isolation but break downstream consumers, older platforms, or dependent services. The author never sees the failure — the consumer does.

## Attack Patterns

1. **signature-diff**: For each changed public signature, enumerate callers across workspace. Verify each caller compiles against the new signature.
2. **removed-symbol-search**: For each removed export/public symbol, search consumers for stale imports.
3. **deprecation-path-check**: For each breaking change, verify deprecation lifecycle (warn → error → remove). New params MUST be optional with defaults. New interface methods MUST have default implementations.
4. **string-externalization**: Scan diff for hardcoded user-facing strings. Flag any not in localization bundle. Verify plural forms handled. Check RTL layout respect.
5. **platform-matrix**: Cross diff against target's supported version matrix. Flag API uses exceeding deployment floor.
6. **accessibility-regression**: For each UI change, check `accessibilityLabel`/`contentDescription`, semantic role, screen-reader navigation order, dynamic-type support.
7. **wire-format-change**: For each serialized type touched, verify back-compat with previously-written records. Flag field removals or type changes without migration.
8. **lsp-contract-preservation**: For each subtype/implementation, verify it honors the base contract. A subtype that throws where base doesn't, returns nil where base guarantees non-nil, or rejects inputs base accepts is a violation.

## Severity

- **BLOCK**: Will fail to compile, link, deserialize, or render for ≥1 known consumer/platform.
- **SHOULD**: Will degrade UX on a supported configuration (a11y regression, missing translation).
- **NIT**: Documentation gap, deprecation note missing, unused export.

## Finding Format

Templates:
- "Changed `fetchUser(id)` to `fetchUser(id, opts)` at {file:line} — caller at {file:line} still passes 1 arg → compile fail."
- "Hardcoded English string 'Submit' at {file:line} — not in localization bundle."
- "Missing `accessibilityLabel` on Button at {file:line} — VoiceOver announces 'button'."
- "API call relies on iOS 18+ method at {file:line} without availability check; floor is iOS 16."
- "Field `userId` removed from `UserDTO` — consumers at {url} still reference it."

## Critical Rules

- A signature-change finding without an enumerated caller set MUST be marked `confidence: low`.
- Visual styling concerns belong in the styling/clean-code lens, not here.
- Wire-format changes without back-compat analysis MUST be flagged BLOCK.
- Semantic accessibility lives HERE. Visual aesthetics live in clean-code lens.
