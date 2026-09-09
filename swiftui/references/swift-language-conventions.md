# Swift Language Conventions (for SwiftUI codebases)

Modern Swift idioms worth enforcing while reviewing or writing SwiftUI code — narrower than a full Swift style guide, focused on the API choices that come up constantly in SwiftUI projects. (Originating source: MIT-licensed, adapted from Paul Hudson's `swiftui-pro` skill.)

## API and formatting choices

- Prefer Swift-native string methods over Foundation equivalents: `replacing("a", with: "b")` not `replacingOccurrences(of: "a", with: "b")`.
- Prefer modern Foundation API: `URL.documentsDirectory` instead of manual `FileManager` directory lookups; `appending(path:)` to append strings to a `URL`.
- Never use C-style number formatting like `String(format: "%.2f", value)`. Use `Text(value, format: .number.precision(.fractionLength(2)))` or another `FormatStyle` API.
- Prefer static member lookup over explicit struct/style instances: `.circle` rather than `Circle()`, `.borderedProminent` rather than `BorderedProminentButtonStyle()`.
- Avoid force unwraps (`!`) and force `try` unless the failure is truly unrecoverable — and even then, prefer `fatalError()` with a clear description over a bare `!`. Prefer `if let`, `guard let`, nil-coalescing, or `try?`/`do-catch`.
- Filter text based on user input with `localizedStandardContains()`, not `contains()` or `localizedCaseInsensitiveContains()`.
- Prefer `Double` over `CGFloat`, except with optionals or `inout` (Swift bridges the two freely everywhere else).
- To count array elements matching a predicate, use `count(where:)` rather than `filter().count`.
- Prefer `Date.now` over `Date()` for clarity.
- `import SwiftUI` already brings in `UIImage`/`NSImage` on the appropriate platform — no need for an explicit `import UIKit`/`import AppKit` just for those types.
- For people's names, prefer `PersonNameComponents` with modern formatting over string interpolation like `Text("\(firstName) \(lastName)")`.
- If a type is repeatedly sorted with the same closure (e.g. `books.sorted { $0.author < $1.author }`), make it conform to `Comparable` so the sort order is centralized.
- Avoid manual date-formatting strings where possible. If manual formatting is required for *user display*, use `"y"` rather than `"yyyy"` so the year is correct in all localizations (this rule doesn't apply to API/data-exchange formats).
- To parse a string into a `Date`, prefer the modern initializer, e.g. `Date(myString, strategy: .iso8601)`.
- Flag silently-swallowed errors from user actions (e.g. `print(error.localizedDescription)` instead of surfacing an alert).
- Prefer `if let value {` shorthand over `if let value = value {`.
- Omit `return` for single-expression functions; use `if`/`switch` as expressions when returning or assigning:

```swift
// Prefer
var tileColor: Color {
    if isCorrect {
        .green
    } else {
        .red
    }
}
```

## Swift Concurrency

- Prefer `async`/`await` over closure-based APIs whenever both are offered.
- Never use Grand Central Dispatch (`DispatchQueue.main.async()`, `DispatchQueue.global()`, etc.) in new code — use `async`/`await`, actors, and `Task` instead.
- Use `Task.sleep(for:)`, never `Task.sleep(nanoseconds:)`.
- Flag mutable shared state that isn't protected by an actor or `@MainActor` (unless the project uses `MainActor` default actor isolation).
- Assume strict concurrency rules apply; flag `@Sendable` violations and data races.
- Before adding `MainActor.run()`, check whether the project's default actor isolation is already `MainActor` — it may be redundant.
- `Task.detached()` is often the wrong call; scrutinize any usage carefully.

For deeper Swift Concurrency migration/review work (actors, `Sendable`, Swift 6 strict-concurrency diagnostics), defer to the dedicated `swift-concurrency` / `swift-concurrency-pro` skills rather than duplicating that ground here.
