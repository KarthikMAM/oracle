# Accessibility

Accessibility is a contract, not a nicety: every interactive element a sighted mouse user can reach, a screen-reader, keyboard, or switch user MUST be able to reach and operate with equal certainty.

## Anchors

- **WCAG 2.2** (W3C, 2023) — Perceivable, Operable, Understandable, Robust; conformance level AA is the floor
- **Apple — Human Interface Guidelines: Accessibility** — `accessibilityLabel`, traits, Dynamic Type, VoiceOver focus
- **Google — Material Design Accessibility & Android `contentDescription`** — TalkBack, 48dp targets, font scaling
- **WAI-ARIA Authoring Practices** (W3C) — roles, names, states; the first rule of ARIA is to use native HTML
- **Inclusive Design Principles** (Swan, Henry et al.) — never rely on a single sense or modality to convey meaning

## Q6.1 — Accessible Name on Every Interactive Element

- **MUST** give every tappable/clickable/focusable/activatable element a non-empty accessible name: `accessibilityLabel` (iOS), `contentDescription` (Android), or visible text / `aria-label` / `aria-labelledby` (web).
- **MUST** describe the action or destination, not the glyph ("Delete message", not "trash icon").
- **MUST** explicitly hide decorative-only images from assistive tech (`accessibilityElementsHidden` / `importantForAccessibility="no"` / `alt=""` / `aria-hidden="true"`).
- **SHOULD** source names from the same string pipeline as visible copy, not hardcoded literals. *Escape hatch: in a single-locale tool, demo, or example snippet a literal is fine; note the intent in a comment.*
- **SHOULD NOT** include the role word the platform already announces ("Submit", not "Submit button"). *Escape hatch: where the platform does not auto-announce the role, the role word MAY be included; note that in a comment.*

### Example

```tsx
// WRONG — icon-only control with no accessible name
<button onClick={onDelete}><TrashIcon /></button>

// RIGHT — prefer visible text; otherwise aria-label, and hide the decorative glyph
<button onClick={onDelete} aria-label={t("delete_message")}>
  <TrashIcon aria-hidden="true" />
</button>
```

## Q6.2 — Semantic Roles and Heading Order

- **MUST** use the native semantic element/role for the job; custom controls MUST declare an explicit role (`role="button"`, `accessibilityRole`, Compose `Modifier.semantics { role = Role.Button }`).
- **MUST** form a logical, non-skipping heading hierarchy (h1 → h2 → h3); levels MUST NOT be chosen for font size.
- **MUST** expose state programmatically — selected, checked, expanded, disabled, busy (`aria-expanded`, `accessibilityValue`, `isSelected` semantics) — not implied by color or position alone.
- **SHOULD** honor the first rule of ARIA: prefer a native element over re-implementing its semantics. *Escape hatch: when no native element fits (e.g., a custom tree-grid), a fully-specified ARIA pattern from the WAI-ARIA Authoring Practices is acceptable, with the pattern named in a code comment.*

### Example

```tsx
// WRONG — div is unreachable by keyboard, has no role, no key handling
<div className="btn" onClick={save}>Save</div>

// RIGHT — native button: focusable, Enter/Space activate, role announced
<button type="button" onClick={save}>Save</button>
```

## Q6.3 — Keyboard, Switch, and Screen-Reader Operability

- **MUST** make every interactive element reachable and operable via keyboard / switch / screen-reader gestures, in a logical order, with no keyboard trap.
- **MUST** follow reading/visual order for focus; positive `tabindex` values MUST NOT be used to re-order — fix DOM/view order instead.
- **MUST** give hover- or drag-only functionality a discrete-activation equivalent (tap, button, or menu action).
- **MUST** handle the platform's activation keys/gestures in custom controls (web: Enter and Space for buttons, Arrow keys for composite widgets).
- **MUST** give a visible, focusable alternative to any gesture or shortcut that is the only path to an action.

### Example

```tsx
// WRONG — action only reachable on mouse hover; keyboard users can't trigger it
<div onMouseEnter={showActions}>{row}</div>

// RIGHT — a focusable button exposes the same action
<div>
  {row}
  <button type="button" onClick={showActions}>Show actions</button>
</div>
```

## Q6.4 — Minimum Touch-Target Size

- **MUST** meet the platform minimum: **44×44 pt** (iOS), **48×48 dp** (Android), **24×24 CSS px** (web, WCAG 2.2 SC 2.5.8).
- **MUST** expand a visually small control's hit area to the minimum via padding, an enlarged frame with a `.contentShape(Rectangle())` (SwiftUI) or an overridden `point(inside:with:)` (UIKit), `minimumTouchTargetSize`, or a transparent expanded touch region — shrinking the visual is fine, shrinking the target is not.
- **SHOULD** space adjacent targets so a tap cannot ambiguously hit two. *Escape hatch: the WCAG exception (inline targets in a sentence, or an equivalent non-overlapping control nearby) MAY be used, documented in a comment.*

### Example

```kotlin
// WRONG — 16.dp icon with a custom gesture detector; pointerInput is NOT covered by
// the automatic 48.dp expansion that clickable() gets, so the target stays under minimum.
Icon(Icons.Default.Close, stringResource(R.string.close),
    Modifier
        .size(16.dp)
        .pointerInput(Unit) { detectTapGestures { dismiss() } })

// RIGHT — visual stays small; target meets 48.dp via sizeIn
Box(
    Modifier
        .sizeIn(minWidth = 48.dp, minHeight = 48.dp)
        .pointerInput(Unit) { detectTapGestures { dismiss() } }
        .semantics { role = Role.Button },
    contentAlignment = Alignment.Center,
) {
    Icon(Icons.Default.Close, stringResource(R.string.close), Modifier.size(16.dp))
}
// Note: a plain Modifier.clickable already auto-expands its touch target to 48.dp via
// LocalMinimumInteractiveComponentSize, so the sizeIn wrapper is only needed for custom
// gesture handlers (pointerInput) or when that enforcement is explicitly disabled.
```

## Q6.5 — Color Contrast and Meaning Beyond Color

- **MUST** meet WCAG AA contrast: **4.5:1** for normal text and **3:1** for large text (≥18 pt regular / ≥14 pt bold) (SC 1.4.3), and **3:1** for UI component / graphical-object boundaries (SC 1.4.11).
- **MUST NOT** use color as the only means of conveying information, indicating an action, prompting a response, or distinguishing a visual element (SC 1.4.1) — pair color with text, an icon, a shape, or an underline.
- **MUST** give disabled, error, selected, and required states each a non-color cue (label, icon, border-style, or text).
- **SHOULD** verify contrast against the actual rendered tokens, not eyeballed. *Escape hatch: purely decorative graphics that convey no information are exempt; mark them so in a comment.*

### Example

```tsx
// WRONG — error conveyed by red border only; invisible to color-blind users
<input className={hasError ? "border-red" : "border-gray"} />

// RIGHT — color PLUS an icon and an associated, announced text message
<input
  aria-invalid={hasError}
  aria-describedby={hasError ? "email-err" : undefined}
  className={hasError ? "border-red" : "border-gray"}
/>
{hasError && (
  <p id="email-err" role="alert">
    <WarningIcon aria-hidden="true" /> Enter a valid email address.
  </p>
)}
```

## Q6.6 — Dynamic Type and Font Scaling

- **MUST** scale text with the OS text-size / Dynamic Type setting: iOS `UIFont.preferredFont` / `.font(.body)` with `adjustsFontForContentSizeCategory`; Android `sp` units (never `dp`/`px` for text); web relative units (`rem`/`em`, never fixed `px` that ignores user zoom).
- **MUST** reflow layouts without clipping or truncation at large content sizes; fixed-height text containers MUST NOT be used. Verify at the largest accessibility size (iOS AX5 / Android 200%).
- **SHOULD NOT** cap text scale below the platform's accessibility sizes. *Escape hatch: if a hard cap is unavoidable (e.g., a fixed-width legacy print template), the cap MUST still allow at least 200% scaling per WCAG SC 1.4.4 and be documented in a comment.*

### Example

```swift
// WRONG — fixed size ignores Dynamic Type; text never scales
Text(title).font(.system(size: 17))

// RIGHT — semantic style scales with the user's setting
Text(title).font(.body)
// UIKit: label.font = .preferredFont(forTextStyle: .body); label.adjustsFontForContentSizeCategory = true
```

## Q6.7 — Focus Management, Motion, and Media

- **MUST** move focus to a sensible target on navigation or major content change (new screen's first heading / back affordance); focus MUST NOT be lost to the document/window root. The change **SHOULD** be announced (`UIAccessibility.post(notification: .screenChanged)`, `announceForAccessibility`, `aria-live`).
- **MUST** trap focus inside modals while open and restore focus to the invoking control on close.
- **MUST** respect the "reduce motion" / "prefers-reduced-motion" setting: non-essential animation, parallax, and auto-playing motion MUST be disabled or replaced with a cross-fade when it is on.
- **MUST** caption pre-recorded video and provide a transcript for audio; non-text content MUST have a text alternative; auto-playing audio/video longer than 3 seconds MUST offer a pause/stop control.

### Example

```swift
// WRONG — pushes a screen but VoiceOver focus stays on the old one
navigationController?.pushViewController(detail, animated: true)

// RIGHT — move/announce focus to the new screen; the argument MUST be a view/element
// that exists on the destination (here the pushed VC's own view), not an invented property.
// Passing detail.view lets VoiceOver pick the new screen's first element.
navigationController?.pushViewController(detail, animated: true)
UIAccessibility.post(notification: .screenChanged, argument: detail.view)
```

## Self-Review Checklist

| # | Check | Required | Rule |
|---|---|---|---|
| 1 | Every interactive element has a non-empty, action-describing accessible name | Required | Q6.1 |
| 2 | Accessible names come from the string pipeline (not hardcoded), or the literal context is documented | Required | Q6.1 |
| 3 | Decorative images explicitly hidden from assistive tech | Required | Q6.1 |
| 4 | Names omit the role word the platform already announces, or the non-auto-announced case is documented | Required | Q6.1 |
| 5 | Native semantic elements/roles used; custom controls declare an explicit role | Required | Q6.2 |
| 6 | Heading hierarchy is logical and non-skipping (not chosen for font size) | Required | Q6.2 |
| 7 | State (selected/checked/expanded/disabled) exposed programmatically | Required | Q6.2 |
| 8 | Every interactive element operable by keyboard/switch/screen-reader, no trap | Required | Q6.3 |
| 9 | Focus order follows visual order; no positive `tabindex` | Required | Q6.3 |
| 10 | Hover-/drag-only actions have a discrete-activation equivalent | Required | Q6.3 |
| 11 | Touch targets meet 44pt iOS / 48dp Android / 24px web minimum | Required | Q6.4 |
| 12 | Small visuals expand hit area to the target minimum | Required | Q6.4 |
| 13 | Adjacent targets have spacing to avoid ambiguous taps, or the WCAG exception is documented | Required | Q6.4 |
| 14 | Text contrast ≥ 4.5:1 normal, ≥ 3:1 large/UI | Required | Q6.5 |
| 15 | No information conveyed by color alone; state has a non-color cue | Required | Q6.5 |
| 16 | Contrast verified against the actual rendered tokens, not eyeballed | Required | Q6.5 |
| 17 | Text scales with Dynamic Type / font-size setting (`sp`/`rem`, semantic styles) | Required | Q6.6 |
| 18 | Layout reflows without clipping/truncation at largest accessibility size | Required | Q6.6 |
| 19 | Focus moves and is announced on navigation; modals trap and restore focus | Required | Q6.7 |
| 20 | Reduce-motion preference respected for non-essential animation | Required | Q6.7 |
| 21 | Media has captions/transcript/alt; long auto-play offers pause/stop | Required | Q6.7 |
