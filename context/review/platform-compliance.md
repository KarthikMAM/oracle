# platform-compliance

Platform-specific rules, SDK version checks, API availability, lifecycle contracts, framework idioms, accessibility, and platform-specific threading.

## Why

Platform issues compile and pass tests in development but fail in production on user devices. iOS apps crash when calling iOS 17 APIs on iOS 15. Android Activities leak when subscriptions outlive the lifecycle. React components flash old state when hooks dependencies are wrong.

## Attack Patterns

1. **api-availability**: Every API call available at minimum deployment target. Flag calls requiring higher version without availability checks (`@available`, `if #available`, `Build.VERSION.SDK_INT`).
2. **lifecycle-contracts**: Components respect platform lifecycle (Activity/Fragment on Android, UIViewController on iOS, mount/unmount in React). Flag state access after destruction or before initialization.
3. **framework-idioms**: Code follows platform-specific patterns (SwiftUI vs UIKit, Compose vs View, hooks rules). Flag deprecated or discouraged patterns.
4. **permission-handling**: Platform capabilities requiring runtime permission (camera, location, notifications) — verify permission check precedes usage and denial is handled gracefully.
5. **accessibility-contract**: UI components have accessibility labels/descriptions, semantic roles, sufficient contrast, minimum touch targets (44×44pt iOS, 48×48dp Android, 24×24px web).
6. **platform-threading**: UI updates on main thread. Heavy work off main thread. Flag platform-specific threading violations (Room DB on main, UIKit from background).
7. **swift-availability-tautology**: When checking iOS version availability, the predicate must reference a higher version than the deployment target, not lower. Flag `if #available(iOS 16, *)` when min target is 17.
8. **android-lifecycle-leak**: Subscriptions, listeners, and observers MUST be unregistered in the appropriate lifecycle method. Flag long-lived registrations without cleanup.
9. **autoSDE-tautology-detection** (when AutoSDE is in pipeline): 6 known false-positive patterns: null-impossible, exhaustively-documented literal, protocol-required parameter, NotNull-annotated, stable-by-construction, type-impossible cast.

## Severity

- **BLOCK**: API used below minimum deployment target without availability check. UI update from background thread. Missing permission check before capability use.
- **SHOULD**: Accessibility metadata missing. Platform anti-pattern that works but degrades UX.
- **NIT**: Non-idiomatic but functional approach.

## Platform Detection

Infer the target platform from:
- File extensions: `.swift` → iOS/macOS, `.kt` → Android/JVM, `.tsx`/`.jsx` → React/Web
- Framework imports: `UIKit`/`SwiftUI`, `android.*`/`androidx.*`, `react`/`next`
- Build config: `deploymentTarget`, `minSdk`, `browserslist`

When platform is ambiguous, ask for clarification via context rather than guessing.

## Finding Format

Templates:
- "Call to `URLSession.data(for:)` at {file:line} requires iOS 15+; deployment target is 14. BLOCK; wrap with `if #available(iOS 15, *)` or use legacy completion API."
- "`view.findViewById()` called in `onCreate` at {file:line} after `setContentView(null)` — Android lifecycle violation. BLOCK."
- "`useState` called inside `if` block at {file:line} — React Rules of Hooks violation. BLOCK."
- "Camera capture started at {file:line} without `Camera` permission check. BLOCK; check permission first."

## Critical Rules

- Findings MUST cite the specific platform/framework rule violated.
- Platform-specific findings MUST identify the affected platform version range.
- Accessibility metadata is a CONTRACT, not a SHOULD — missing labels on interactive elements is BLOCK on UI code.
