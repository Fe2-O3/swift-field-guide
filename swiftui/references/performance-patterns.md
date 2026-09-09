# SwiftUI Performance Patterns Reference

## Table of Contents

- [Performance Optimization](#performance-optimization)
- [Anti-Patterns](#anti-patterns)
- [Summary Checklist](#summary-checklist)

## Performance Optimization

### 1. Avoid Redundant State Updates

SwiftUI doesn't compare values before triggering updates:

```swift
// BAD - triggers update even if value unchanged
.onReceive(publisher) { value in
    self.currentValue = value  // Always triggers body re-evaluation
}

// GOOD - only update when different
.onReceive(publisher) { value in
    if self.currentValue != value {
        self.currentValue = value
    }
}
```

### 2. Optimize Hot Paths

Hot paths are frequently executed code (scroll handlers, animations, gestures):

```swift
// BAD - updates state on every scroll position change
.onPreferenceChange(ScrollOffsetKey.self) { offset in
    shouldShowTitle = offset.y <= -32  // Fires constantly during scroll!
}

// GOOD - only update when threshold crossed
.onPreferenceChange(ScrollOffsetKey.self) { offset in
    let shouldShow = offset.y <= -32
    if shouldShow != shouldShowTitle {
        shouldShowTitle = shouldShow  // Fires only when crossing threshold
    }
}
```

### 3. Pass Only What Views Need

**Avoid passing large "config" or "context" objects.** Pass only the specific values each view needs.

```swift
// Good - pass specific values
ThemeSelector(theme: config.theme)
FontSizeSlider(fontSize: config.fontSize)

// Avoid - passing entire config (creates broad dependency)
ThemeSelector(config: config)  // Notified of ALL config changes
```

With `ObservableObject`, any `@Published` change triggers all observers. With `@Observable`, views update only when accessed properties change, but passing entire objects still creates broader dependencies than necessary.

### 4. Use Equatable Views

For views with expensive bodies, conform to `Equatable`:

```swift
struct ExpensiveView: View, Equatable {
    let data: SomeData

    static func == (lhs: Self, rhs: Self) -> Bool {
        lhs.data.id == rhs.data.id  // Custom equality check
    }

    var body: some View {
        // Expensive computation
    }
}

// Usage
ExpensiveView(data: data)
    .equatable()  // Use custom equality
```

**Caution**: If you add new state or dependencies to your view, remember to update your `==` function!

### 5. POD Views for Fast Diffing

**POD (Plain Old Data) views use `memcmp` for fastest diffing.** A view is POD if it only contains simple value types and no property wrappers.

```swift
// POD view - fastest diffing
struct FastView: View {
    let title: String
    let count: Int
    
    var body: some View {
        Text("\(title): \(count)")
    }
}

// Non-POD view - uses reflection or custom equality
struct SlowerView: View {
    let title: String
    @State private var isExpanded = false  // Property wrapper makes it non-POD
    
    var body: some View {
        Text(title)
    }
}
```

**Advanced Pattern**: Wrap expensive non-POD views in POD parent views:

```swift
// POD wrapper for fast diffing
struct ExpensiveView: View {
    let value: Int
    
    var body: some View {
        ExpensiveViewInternal(value: value)
    }
}

// Internal view with state
private struct ExpensiveViewInternal: View {
    let value: Int
    @State private var item: Item?
    
    var body: some View {
        // Expensive rendering
    }
}
```

**Why**: The POD parent uses fast `memcmp` comparison. Only when `value` changes does the internal view get diffed.

### 6. Lazy Loading

Use lazy containers for large collections:

```swift
// BAD - creates all views immediately
ScrollView {
    VStack {
        ForEach(items) { item in
            ExpensiveRow(item: item)
        }
    }
}

// GOOD - creates views on demand
ScrollView {
    LazyVStack {
        ForEach(items) { item in
            ExpensiveRow(item: item)
        }
    }
}
```

**iOS 26+ note**: Nested scroll views containing lazy stacks now automatically defer loading their children until they are about to appear, matching the behavior of top-level lazy stacks. This benefits patterns like horizontal photo carousels inside a vertical scroll view.

> Source: "What's new in SwiftUI" (WWDC25, session 256)

### 7. Task Cancellation

Cancel async work when view disappears:

```swift
struct DataView: View {
    @State private var data: [Item] = []

    var body: some View {
        List(data) { item in
            Text(item.name)
        }
        .task {
            // Automatically cancelled when view disappears
            data = await fetchData()
        }
    }
}
```

### 8. Debug View Updates

**Use `Self._printChanges()` or `Self._logChanges()` to debug unexpected view updates.**

```swift
struct DebugView: View {
    @State private var count = 0
    @State private var name = ""
    
    var body: some View {
        #if DEBUG
        let _ = Self._logChanges()  // Xcode 15.1+: logs to com.apple.SwiftUI subsystem
        #endif
        
        VStack {
            Text("Count: \(count)")
            Text("Name: \(name)")
        }
    }
}
```

- `Self._printChanges()`: Prints which properties changed to standard output.
- `Self._logChanges()` (iOS 17+): Logs to the `com.apple.SwiftUI` subsystem with category "Changed Body Properties", using `os_log` for structured output.

Both print `@self` when the view value itself changed and `@identity` when the view's persistent data was recycled.

**Why**: This helps identify which state changes are causing view updates. Isolating redraw triggers into single-responsibility subviews is often the fix -- extracting a subview means SwiftUI can skip its body when its inputs haven't changed.

### 9. Eliminate Unnecessary Dependencies

**Narrow state scope to reduce update fan-out.** Instead of passing an entire `@Observable` model to a row view (which creates a dependency on all accessed properties), pass only the specific values the view needs as `let` properties.

```swift
// Bad - broad dependency on entire model
struct ItemRow: View {
    @Environment(AppModel.self) private var model
    let item: Item
    var body: some View { Text(item.name).foregroundStyle(model.theme.primaryColor) }
}

// Good - narrow dependency
struct ItemRow: View {
    let item: Item
    let themeColor: Color
    var body: some View { Text(item.name).foregroundStyle(themeColor) }
}
```

**Avoid storing frequently-changing values in the environment.** Even when a view doesn't read the changed key, SwiftUI still checks all environment readers. This cost adds up with many views and frequent updates (geometry values, timers).

> Source: "Optimize SwiftUI performance with Instruments" (WWDC25, session 306)

### 10. @Observable Dependency Granularity

**Consider per-item `@Observable` state holders (one per row/item) to narrow update scope.** When multiple list items share a dependency on the same `@Observable` array, changing one element causes all items to re-evaluate their bodies.

```swift
// BAD - all item views depend on the full favorites array
@Observable
class ModelData {
    var favorites: [Landmark] = []

    func isFavorite(_ landmark: Landmark) -> Bool {
        favorites.contains(landmark)
    }
}

struct LandmarkRow: View {
    let landmark: Landmark
    @Environment(ModelData.self) private var model

    var body: some View {
        HStack {
            Text(landmark.name)
            if model.isFavorite(landmark) {
                Image(systemName: "heart.fill")
            }
        }
    }
}

// GOOD - each item has its own observable view model
@Observable
class LandmarkViewModel {
    var isFavorite: Bool = false
}

struct LandmarkRow: View {
    let landmark: Landmark
    let viewModel: LandmarkViewModel

    var body: some View {
        HStack {
            Text(landmark.name)
            if viewModel.isFavorite {
                Image(systemName: "heart.fill")
            }
        }
    }
}
```

**Why**: With the bad pattern, toggling one favorite marks the entire array as changed, causing every `LandmarkRow` to re-run its body. With per-item view models, only the toggled item's body runs.

> Source: "Optimize SwiftUI performance with Instruments" (WWDC25, session 306)

### 11. Off-Main-Thread Closures

**SwiftUI may call certain closures on a background thread for performance.** These closures must be `Sendable` and should avoid accessing `@MainActor`-isolated state directly. Instead, capture needed values in the closure's capture list.

Closures that may run off the main thread:
- `Shape.path(in:)`
- `visualEffect` closure
- `Layout` protocol methods
- `onGeometryChange` transform closure

```swift
// BAD - accessing @MainActor state directly
.visualEffect { content, geometry in
    content.blur(radius: self.pulse ? 5 : 0)  // Compiler error: @MainActor isolated
}

// GOOD - capture the value
.visualEffect { [pulse] content, geometry in
    content.blur(radius: pulse ? 5 : 0)
}
```

> Source: "Explore concurrency in SwiftUI" (WWDC25, session 266)

### 12. Common Performance Issues

**Be aware of common performance bottlenecks in SwiftUI:**

- View invalidation storms from broad state changes
- Unstable identity in lists causing excessive diffing
- Heavy work in `body` (formatting, sorting, image decoding)
- Layout thrash from deep stacks or preference chains

**When performance issues arise**, suggest the user profile with Instruments (SwiftUI template) to identify specific bottlenecks.

## Anti-Patterns

### 1. Creating Objects in Body

```swift
// BAD - creates new formatter every body call
var body: some View {
    let formatter = DateFormatter()
    formatter.dateStyle = .long
    return Text(formatter.string(from: date))
}

// GOOD - static or stored formatter
private static let dateFormatter: DateFormatter = {
    let f = DateFormatter()
    f.dateStyle = .long
    return f
}()

var body: some View {
    Text(Self.dateFormatter.string(from: date))
}
```

### 2. Heavy Computation in Body

**Keep view body simple and pure.** Avoid side effects, dispatching, or complex logic.

```swift
// BAD - sorts array every body call
var body: some View {
    List(items.sorted { $0.name < $1.name }) { item in Text(item.name) }
}

// GOOD - compute once, update via onChange or a computed property in the model
@State private var sortedItems: [Item] = []

var body: some View {
    List(sortedItems) { item in Text(item.name) }
        .onChange(of: items) { _, newItems in
            sortedItems = newItems.sorted { $0.name < $1.name }
        }
}
```

Move sorting, filtering, and formatting into models or computed properties. The `body` should be a pure structural representation of state.

### 3. Unnecessary State

```swift
// BAD - derived state stored separately
@State private var items: [Item] = []
@State private var itemCount: Int = 0  // Unnecessary!

// GOOD - compute derived values
@State private var items: [Item] = []

var itemCount: Int { items.count }  // Computed property
```

## Additional Performance Patterns

### 13. Ternary Over if/else for Modifier Toggling

When toggling a modifier's value based on a condition, prefer a ternary expression over `if`/`else` view branching. Branching wraps the content in `_ConditionalContent`, which breaks structural identity and can cause SwiftUI to recreate the underlying platform view instead of updating it in place.

```swift
// Prefer
.opacity(isVisible ? 1 : 0)

// Over (when the branches are otherwise identical)
if isVisible {
    content.opacity(1)
} else {
    content.opacity(0)
}
```

### 14. Avoid AnyView

Avoid `AnyView` unless absolutely required — it erases type information SwiftUI uses for diffing. Use `@ViewBuilder`, `Group`, or generics instead. (See also `AnyView` coverage further up this file for the type-checking-performance angle.)

### 15. Static ScrollView Backgrounds

If a `ScrollView` has an opaque, static, solid background, use `.scrollContentBackground(.visible)` to improve scroll-edge rendering efficiency rather than leaving the default.

### 16. Prefer Formatted Text Over Formatter Properties

Where possible, skip storing a `DateFormatter`/`NumberFormatter` property altogether and use `Text` with a format style: `Text(Date.now, format: .dateTime.day().month().year())` or `Text(100, format: .currency(code: "USD"))`.

### 17. Avoid Repeated Inline Transforms in ForEach/List

Avoid expensive inline transforms in `List`/`ForEach` initializers (`items.filter { ... }`) when they run often. Derive transformed data from the source of truth with a `let`/computed property, or cache in `@State` — but only cache derived collections in `@State` if you also own explicit invalidation logic, to avoid stale UI.

### 18. Store Built Views, Not Escaping @ViewBuilder Closures

```swift
// Anti-pattern: stores an escaping closure on the view.
struct CardView<Content: View>: View {
    let content: () -> Content

    var body: some View {
        VStack(alignment: .leading) {
            content()
        }
        .padding()
        .background(.ultraThinMaterial)
        .clipShape(.rect(cornerRadius: 8))
    }
}

// Preferred: store the built view value; the synthesized init handles calling the builder.
struct CardView<Content: View>: View {
    @ViewBuilder let content: Content

    var body: some View {
        VStack(alignment: .leading) {
            content
        }
        .padding()
        .background(.ultraThinMaterial)
        .clipShape(.rect(cornerRadius: 8))
    }
}
```

### 19. Debounced Search

Cancel and restart the previous search `Task` rather than firing a request per keystroke:

```swift
@Observable
@MainActor
final class SearchViewModel {
    var searchText = ""
    var results: [Article] = []

    private var searchTask: Task<Void, Never>?

    func updateSearch(_ text: String) {
        searchText = text

        searchTask?.cancel()
        searchTask = Task {
            try? await Task.sleep(for: .milliseconds(300))

            guard !Task.isCancelled else { return }

            results = await performSearch(text)
        }
    }

    private func performSearch(_ query: String) async -> [Article] {
        // Search logic
        []
    }
}
```

### 20. Pagination via onAppear

Trigger the next page load when the last row appears, on top of `LazyVStack`/`LazyVGrid`:

```swift
LazyVStack(spacing: 16) {
    ForEach(articles) { article in
        ArticleCard(article: article)
            .onAppear {
                if article == articles.last {
                    loadMoreArticles()
                }
            }
    }
}
```

## Instruments 26 SwiftUI Template — Diagnostic Protocol

This is a lighter-weight, built-in-Xcode complement to the `.trace`-file analysis workflow (`references/trace-analysis.md`/`references/trace-recording.md` and `scripts/analyze_trace.py`) — reach for it first for a quick look, and the scripted analysis when you need to correlate against source or scope a specific window.

**Two problems this instrument surfaces:**

1. **Long View Body Updates** — a body takes too long to evaluate.
2. **Unnecessary Updates** — views update when their data hasn't actually changed.

**Workflow:**

1. Press **Cmd-I** in Xcode, choose the **SwiftUI** template.
2. Check the **Long View Body Updates** lane (red = priority).
3. For unnecessary updates specifically, watch for patterns like:
   - **Shared dependencies** — a view reading a whole collection (e.g. `favorites.contains(item)`) re-renders whenever *any* element changes. Prefer per-item `@Observable` view models keyed by ID so each row only depends on its own state.
   - **High-frequency environment values** — writing to `.environment(\.scrollOffset, offset)` 60x/second invalidates every reader. Pass such values directly to the specific child that needs them instead.

**iOS 26 automatic wins** (from rebuilding with the iOS 26 SDK, no code changes required): up to 6x faster list loading for 100k+ item lists, up to 16x faster list updates, reduced dropped frames, and nested `ScrollView` lazy loading (see item 6 above).

**30-minute diagnostic protocol** for a reported hitch/hang:

| Step | Time |
|------|------|
| Build Release | 5 min |
| Trigger issue | 3 min |
| Record trace | 5 min |
| Review Long Updates | 5 min |
| Check Cause & Effect | 5 min |
| Identify view | 2 min |

**Before shipping a fix**, confirm:
- [ ] Ran the SwiftUI Instrument?
- [ ] Know which view is expensive?
- [ ] Can explain *why* the fix helps?
- [ ] Verified the improvement in Instruments (not just "feels faster")?

## Summary Checklist

- [ ] State updates check for value changes before assigning
- [ ] Hot paths minimize state updates
- [ ] Pass only needed values to views (avoid large config objects)
- [ ] Large lists use `LazyVStack`/`LazyHStack`
- [ ] No object creation in `body`
- [ ] Heavy computation moved out of `body`
- [ ] Body kept simple and pure (no side effects)
- [ ] Derived state computed, not stored
- [ ] Use `Self._logChanges()` or `Self._printChanges()` to debug unexpected updates
- [ ] Equatable conformance for expensive views (when appropriate)
- [ ] Consider POD view wrappers for advanced optimization
- [ ] Consider using granular @Observable dependencies for list items (smaller observable units per row when it measurably reduces updates)
- [ ] Frequently-changing values not stored in the environment
- [ ] Sendable closures (Shape, visualEffect, Layout) capture values instead of accessing @MainActor state
