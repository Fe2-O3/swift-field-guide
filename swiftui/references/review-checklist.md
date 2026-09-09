# Comprehensive Review Workflow

A structured process for reviewing SwiftUI/Swift code end-to-end: correctness, modern API usage, maintainability, and performance. Use this when the user asks for a full review rather than help with one specific topic. (Originating source: MIT-licensed, adapted from Paul Hudson's `swiftui-pro` skill — see attribution note at the end.)

Report only genuine problems — do not nitpick or invent issues.

## Review process

1. Check for deprecated API using `references/latest-apis.md`.
2. Check that views, modifiers, and animations are written optimally using `references/view-structure.md` and `references/animation-basics.md` / `animation-transitions.md` / `animation-advanced.md`.
3. Validate that data flow is configured correctly using `references/state-management.md`.
4. Ensure navigation is updated and performant using `references/sheet-navigation-patterns.md`.
5. Ensure the code uses designs that are accessible and compliant with Apple's Human Interface Guidelines using `references/design-and-hig.md`.
6. Validate accessibility compliance (Dynamic Type, VoiceOver, Reduce Motion) using `references/accessibility-patterns.md`.
7. Ensure the code runs efficiently using `references/performance-patterns.md`.
8. Quick validation of Swift code using `references/swift-language-conventions.md`.
9. Final code hygiene check using the Hygiene section below.

If doing a partial review, load only the relevant reference files — this list is a menu, not a mandate to load everything for a one-line fix.

## Core Instructions

- iOS 26 exists and is the default deployment target for new apps.
- Target Swift 6.2 or later, using modern Swift concurrency (see `references/swift-language-conventions.md`).
- As a SwiftUI developer, avoid UIKit/AppKit unless requested or genuinely necessary (see `references/uikit-interop.md`).
- Do not introduce third-party frameworks without asking first.
- Break different types into different Swift files rather than placing multiple structs/classes/enums in one file.
- Use a consistent project structure, with folder layout determined by app features.

## Hygiene

- Never include secrets (API keys, tokens) in the repository.
- Add code/doc comments where the logic isn't self-evident.
- Unit tests should exist for core application logic; UI tests only where unit tests aren't possible.
- `@AppStorage` must never store usernames, passwords, or other sensitive data — use the Keychain (see also the `@ObservationIgnored @AppStorage` caveat in `references/state-management.md`).
- If SwiftLint is configured, it should return no warnings or errors.
- If the project uses `Localizable.xcstrings`, prefer adding user-facing strings via symbol keys (e.g. `helloWorld`) with `extractionState` set to `"manual"`, accessed via generated symbols like `Text(.helloWorld)`. Offer to translate new keys into all languages the project supports.
- If an Xcode MCP is configured, prefer its tools over generic alternatives (e.g. a preview-rendering tool to capture SwiftUI preview images, a documentation-search tool for Apple API lookups).

## Output Format

Organize findings by file. For each issue:

1. State the file and relevant line(s).
2. Name the rule being violated (e.g., "Use `foregroundStyle()` instead of `foregroundColor()`").
3. Show a brief before/after code fix.

Skip files with no issues. End with a prioritized summary of the most impactful changes to make first.

### Example output

#### ContentView.swift

**Line 12: Use `foregroundStyle()` instead of `foregroundColor()`.**

```swift
// Before
Text("Hello").foregroundColor(.red)

// After
Text("Hello").foregroundStyle(.red)
```

**Line 24: Icon-only button is bad for VoiceOver - add a text label.**

```swift
// Before
Button(action: addUser) {
    Image(systemName: "plus")
}

// After
Button("Add User", systemImage: "plus", action: addUser)
```

**Line 31: Avoid `Binding(get:set:)` in view body - use `@State` with `onChange()` instead.**

```swift
// Before
TextField("Username", text: Binding(
    get: { model.username },
    set: { model.username = $0; model.save() }
))

// After
TextField("Username", text: $model.username)
    .onChange(of: model.username) {
        model.save()
    }
```

#### Summary

1. **Accessibility (high):** The add button on line 24 is invisible to VoiceOver.
2. **Deprecated API (medium):** `foregroundColor()` on line 12 should be `foregroundStyle()`.
3. **Data flow (medium):** The manual binding on line 31 is fragile and harder to maintain.

*End of example.*

## Related: this skill's own Correctness Checklist

For the always-a-bug checklist (private `@State`, `ForEach` identity, `.animation(_:value:)`, etc.) used during any review — not just a comprehensive one — see the Correctness Checklist in `SKILL.md`.

---

**Attribution:** the review process, output format, and hygiene checklist above, along with `references/design-and-hig.md` and `references/swift-language-conventions.md`, originate from Paul Hudson's `swiftui-pro` Agent Skill (MIT License). They were folded into this merged skill rather than kept as a separate near-duplicate skill; the license terms travel with the content.
