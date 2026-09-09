# Design and Human Interface Guidelines Compliance

Guidance for building SwiftUI designs that are visually uniform, flexible across devices/Dynamic Type, and compliant with Apple's Human Interface Guidelines. (Originating source: MIT-licensed, adapted from Paul Hudson's `swiftui-pro` skill.)

## Creating a uniform design in an app

Prefer to place standard fonts, sizes, colors, stack spacing, padding, rounding, and animation timings into a shared enum of constants, so they can be reused across all views. This keeps the app's design consistent and lets it be adjusted from one place.

## Requirements for flexible, accessible design

- Never use `UIScreen.main.bounds` to read available space; prefer `containerRelativeFrame()`, `visualEffect()`, or (only if there's no alternative) `GeometryReader` — see `references/gestures-and-layout.md` for the adaptive-layout decision tree.
- Avoid fixed frames for views unless content is guaranteed to fit neatly inside; fixed sizing causes problems across device sizes and Dynamic Type settings. Prefer flexible frames.
- Apple's minimum acceptable tap target on iOS is 44x44 points. Enforce this strictly for any custom tappable control.

## Standard system styling

- Prefer `ContentUnavailableView` over a custom-built empty/missing-data state.
- With `searchable()`, `ContentUnavailableView.search` already includes the search term automatically — don't pass `ContentUnavailableView.search(text: searchText)`.
- For icon + text laid out horizontally, prefer `Label` over a manual `HStack`.
- Prefer system hierarchical styles (`.secondary`, `.tertiary`, etc.) over manual opacity, so the system adapts to context (dark mode, accessibility settings) automatically.
- Inside `Form`, wrap controls like `Slider` in `LabeledContent` so title and control lay out correctly. `LabeledContent` also works outside `Form` for any title-value display — consider a custom `LabeledContentStyle` for consistent layout across views.
- `RoundedRectangle`'s default rounding style is already `.continuous` — no need to specify it explicitly.

## Ensuring designs work for everyone

- Use `bold()` instead of `.fontWeight(.bold)` — `bold()` lets the system choose the correct weight for the current context (Dynamic Type, accessibility bold text, etc.).
- Only use `fontWeight()` for non-bold weights when there's a specific reason; scattering `.medium`/`.semibold` around is counterproductive.
- Avoid hard-coded padding/spacing values unless specifically requested.
- Avoid `UIColor` in SwiftUI code; use SwiftUI `Color` or asset-catalog colors.
- `.caption2` is extremely small and generally best avoided; even `.caption` is on the small side — use both carefully.

For the accessibility-specific checklist (VoiceOver, Dynamic Type, Reduce Motion, Differentiate Without Color), see `references/accessibility-patterns.md`.
