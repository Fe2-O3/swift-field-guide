# SwiftUI State Management Reference

## Table of Contents

- [Property Wrapper Selection Guide](#property-wrapper-selection-guide)
- [@State](#state)
- [Property Wrappers Inside @Observable Classes](#property-wrappers-inside-observable-classes)
- [@Binding](#binding)
- [@FocusState](#focusstate)
- [@StateObject vs @ObservedObject (Legacy - Pre-iOS 17)](#stateobject-vs-observedobject-legacy---pre-ios-17)
- [Don't Pass Values as @State](#dont-pass-values-as-state)
- [@Bindable (iOS 17+)](#bindable-ios-17)
- [let vs var for Passed Values](#let-vs-var-for-passed-values)
- [Environment and Preferences](#environment-and-preferences)
- [Decision Flowchart](#decision-flowchart)
- [State Privacy Rules](#state-privacy-rules)
- [Avoid Nested ObservableObject](#avoid-nested-observableobject)
- [Key Principles](#key-principles)

## Property Wrapper Selection Guide

| Wrapper | Use When | Notes |
|---------|----------|-------|
| `@State` | Internal view state that triggers updates | Must be `private` |
| `@Binding` | Child view needs to modify parent's state | Don't use for read-only |
| `@Bindable` | iOS 17+: View receives `@Observable` object and needs bindings | For injected observables |
| `let` | Read-only value passed from parent | Simplest option |
| `var` | Read-only value that child observes via `.onChange()` | For reactive reads |

**Legacy (Pre-iOS 17):**
| Wrapper | Use When | Notes |
|---------|----------|-------|
| `@StateObject` | View owns an `ObservableObject` instance | Use `@State` with `@Observable` instead |
| `@ObservedObject` | View receives an `ObservableObject` from outside | Never create inline |

## @State

Always mark `@State` properties as `private`. Use for internal view state that triggers UI updates.

```swift
// Correct
@State private var isAnimating = false
@State private var selectedTab = 0
```

**Why Private?** Marking state as `private` makes it clear what's created by the view versus what's passed in. It also prevents accidentally passing initial values that will be ignored (see "Don't Pass Values as @State" below).

### iOS 17+ with @Observable (Preferred)

**Always prefer `@Observable` over `ObservableObject`.** With iOS 17's `@Observable` macro, use `@State` instead of `@StateObject`:

```swift
@Observable
@MainActor  // Always mark @Observable classes with @MainActor
final class DataModel {
    var name = "Some Name"
    var count = 0
}

struct MyView: View {
    @State private var model = DataModel()  // Use @State, not @StateObject

    var body: some View {
        VStack {
            TextField("Name", text: $model.name)
            Stepper("Count: \(model.count)", value: $model.count)
        }
    }
}
```

**Critical**: When a view *owns* an `@Observable` object, always use `@State` -- not `let`. Without `@State`, SwiftUI may recreate the instance when a parent view redraws, losing accumulated state. `@State` tells SwiftUI to preserve the instance across view redraws. Using `@State` also provides bindings directly (no need for `@Bindable`).

**Note**: You may want to mark `@Observable` classes with `@MainActor` to ensure thread safety with SwiftUI, unless your project or package uses Default Actor Isolation set to `MainActor`—in which case, the explicit attribute is redundant and can be omitted.

## Property Wrappers Inside @Observable Classes

**Critical**: The `@Observable` macro transforms stored properties to add observation tracking. Property wrappers (like `@AppStorage`, `@SceneStorage`, `@Query`) also transform properties with their own storage. These two transformations conflict, causing a compiler error.

**Always annotate property-wrapper properties with `@ObservationIgnored` inside `@Observable` classes.**

```swift
@Observable
@MainActor
final class SettingsModel {
    // WRONG - compiler error: property wrappers conflict with @Observable
    // @AppStorage("username") var username = ""

    // CORRECT - @ObservationIgnored prevents the conflict
    @ObservationIgnored @AppStorage("username") var username = ""
    @ObservationIgnored @AppStorage("isDarkMode") var isDarkMode = false

    // Regular stored properties work fine with @Observable
    var isLoading = false
}
```

This applies to **any** property wrapper used inside an `@Observable` class, including but not limited to:
- `@AppStorage`
- `@SceneStorage`
- `@Query` (SwiftData)

**Note**: Since `@ObservationIgnored` disables observation tracking for that property, SwiftUI won't detect changes through the Observation framework. However, property wrappers like `@AppStorage` already notify SwiftUI of changes through their own mechanisms (e.g., UserDefaults KVO), so views still update correctly.

**Never remove `@ObservationIgnored`** from property-wrapper properties in `@Observable` classes — doing so causes a compiler error.

**Flagged conflict — does `@ObservationIgnored @AppStorage` still update the view?** One merged source claimed `@AppStorage` "already notifies SwiftUI of changes through their own mechanisms (e.g., UserDefaults KVO), so views still update correctly" even when marked `@ObservationIgnored`. Another source states flatly: never rely on `@AppStorage` inside an `@Observable` class to trigger view updates, even with `@ObservationIgnored`. These disagree, and the second is the technically defensible one: `@AppStorage`'s live-update behavior comes from SwiftUI calling `DynamicProperty.update()` on properties declared directly on a `View` struct — that mechanism is never invoked for a property wrapper stored inside a plain (non-`View`) class, `@Observable` or not. Treat `@ObservationIgnored @AppStorage` inside a model class as inert for observation purposes: it persists to `UserDefaults`, but nothing will re-render a view because of it. If you need both persistence and reactivity, read once at init and re-publish through a regular observed property, or keep the `@AppStorage` declaration directly on the `View`.

**Also never use `@AppStorage` for secrets.** Usernames, passwords, tokens, or any sensitive data must go in the Keychain — `@AppStorage`/`UserDefaults` is unencrypted plist storage.

## @Binding

Use only when child view needs to **modify** parent's state. If child only reads the value, use `let` instead.

```swift
// Parent
struct ParentView: View {
    @State private var isSelected = false

    var body: some View {
        ChildView(isSelected: $isSelected)
    }
}

// Child - will modify the value
struct ChildView: View {
    @Binding var isSelected: Bool

    var body: some View {
        Button("Toggle") {
            isSelected.toggle()
        }
    }
}
```

### When NOT to use @Binding

- **Don't use `@Binding` for read-only values.** If the child only displays the value and never modifies it, use `let` instead. `@Binding` adds unnecessary overhead and implies a write contract that doesn't exist.

### Don't over-use @Bindable either

The same idea applies to `@Bindable`: reach for it only where the view actually needs a two-way binding into a mutable property. Wrapping a passed-in `@Observable` model in `@Bindable` "just in case" — including for read-only computed properties — causes unnecessary observation and view reloads. If the view only reads, a plain (non-`@Bindable`) reference is enough.

## @FocusState

See `references/focus-patterns.md` for comprehensive focus management guidance including `@FocusState`, `@FocusedValue`, `.focusable()`, default focus, and common pitfalls.

Always mark `@FocusState` as `private`.

## @StateObject vs @ObservedObject (Legacy - Pre-iOS 17)

**Note**: Always prefer `@Observable` with `@State` for iOS 17+.

The key distinction is **ownership**: `@StateObject` when the view **creates and owns** the object; `@ObservedObject` when the view **receives** it from outside.

```swift
// View creates it → @StateObject
@StateObject private var viewModel = MyViewModel()

// View receives it → @ObservedObject
@ObservedObject var viewModel: MyViewModel
```

**Never** create an `ObservableObject` inline with `@ObservedObject` -- it recreates the instance on every view update.

### @StateObject instantiation in View's initializer

Prefer storing the `@StateObject` in the parent view and passing it down. If you must create one in a custom initializer, pass the expression directly to `StateObject(wrappedValue:)` so the `@autoclosure` prevents redundant allocations:

```swift
// Inside a View's init(movie:):
// WRONG — assigning to a local first defeats @autoclosure
let vm = MovieDetailsViewModel(movie: movie)
_viewModel = StateObject(wrappedValue: vm)

// CORRECT — inline expression defers creation
_viewModel = StateObject(wrappedValue: MovieDetailsViewModel(movie: movie))
```

**Modern Alternative**: Use `@Observable` with `@State` instead.

## Don't Pass Values as @State

**Critical**: Never declare passed values as `@State` or `@StateObject`. They only accept an initial value and ignore subsequent updates from the parent.

```swift
// WRONG - child ignores parent updates
struct ChildView: View {
    @State var item: Item  // Shows initial value forever!
    var body: some View { Text(item.name) }
}

// CORRECT - child receives updates
struct ChildView: View {
    let item: Item  // Or @Binding if child needs to modify
    var body: some View { Text(item.name) }
}
```

**Prevention**: Always mark `@State` and `@StateObject` as `private`. This prevents them from appearing in the generated initializer.

## @Bindable (iOS 17+)

Use when receiving an `@Observable` object from outside and needing bindings:

```swift
@Observable
final class UserModel {
    var name = ""
    var email = ""
}

struct ParentView: View {
    @State private var user = UserModel()

    var body: some View {
        EditUserView(user: user)
    }
}

struct EditUserView: View {
    @Bindable var user: UserModel  // Received from parent, needs bindings

    var body: some View {
        Form {
            TextField("Name", text: $user.name)
            TextField("Email", text: $user.email)
        }
    }
}
```

## let vs var for Passed Values

### Use `let` for read-only display

```swift
struct ProfileHeader: View {
    let username: String
    let avatarURL: URL

    var body: some View {
        HStack {
            AsyncImage(url: avatarURL)
            Text(username)
        }
    }
}
```

### Use `var` when reacting to changes with `.onChange()`

```swift
struct ReactiveView: View {
    var externalValue: Int  // Watch with .onChange()
    @State private var displayText = ""

    var body: some View {
        Text(displayText)
            .onChange(of: externalValue) { oldValue, newValue in
                displayText = "Changed from \(oldValue) to \(newValue)"
            }
    }
}
```

## Environment and Preferences

### @Environment

Access environment values provided by SwiftUI or parent views:

```swift
struct MyView: View {
    @Environment(\.colorScheme) private var colorScheme
    @Environment(\.dismiss) private var dismiss

    var body: some View {
        Button("Done") { dismiss() }
            .foregroundStyle(colorScheme == .dark ? .white : .black)
    }
}
```

### Custom Environment Values with @Entry

Use the `@Entry` macro (Xcode 16+, backward compatible to iOS 13) to define custom environment values without boilerplate:

```swift
extension EnvironmentValues {
    @Entry var accentTheme: Theme = .default
}

// Inject
ContentView()
    .environment(\.accentTheme, customTheme)

// Access
struct ThemedView: View {
    @Environment(\.accentTheme) private var theme
}
```

The `@Entry` macro replaces the manual `EnvironmentKey` conformance pattern. It also works with `TransactionValues`, `ContainerValues`, and `FocusedValues`.

### @Environment with @Observable (iOS 17+ - Preferred)

**Always prefer this pattern** for sharing state through the environment:

```swift
@Observable
@MainActor
final class AppState {
    var isLoggedIn = false
}

// Inject
ContentView()
    .environment(AppState())

// Access
struct ChildView: View {
    @Environment(AppState.self) private var appState
}
```

### @EnvironmentObject (Legacy - Pre-iOS 17)

Legacy pattern: inject with `.environmentObject(AppState())`, access with `@EnvironmentObject var appState: AppState`. Prefer `@Observable` with `@Environment` instead.

### Why @Observable over ObservableObject — the short version

- **Less boilerplate** — no `@Published` needed; every stored property is tracked automatically.
- **Fine-grained observation** — a view only re-renders when a property it actually *read* changes, not on every change to the object (unlike `ObservableObject`, which republishes on any `@Published` change regardless of what the view reads).
- **Type-safe environment** — `@Environment(Type.self)` replaces the stringly-untyped-feeling `@EnvironmentObject`.
- **Simpler bindings** — `@Bindable` replaces `@ObservedObject` for injected models that need two-way bindings.
- **Nesting works** — `@Observable` objects nested inside other `@Observable` objects are tracked correctly; `ObservableObject` cannot do this (see "Avoid Nested ObservableObject" below).

### App-level environment injection

Injecting shared app state is typically done once, at the `WindowGroup` / root scene level:

```swift
@Observable
@MainActor
final class AppSettings {
    var isDarkMode = false
}

struct MyApp: App {
    @State private var settings = AppSettings()

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environment(settings)
        }
    }
}

struct SettingsView: View {
    @Environment(AppSettings.self) private var settings

    var body: some View {
        Toggle("Dark Mode", isOn: $settings.isDarkMode)
    }
}
```

## Worked Examples

### Form with validation

```swift
@Observable
class FormModel {
    var email: String = ""
    var isValid: Bool { email.contains("@") }
}

struct FormView: View {
    @State private var model = FormModel()

    var body: some View {
        Form {
            TextField("Email", text: $model.email)
            Button("Submit") { }
                .disabled(!model.isValid)
        }
    }
}
```

### Loading state with `.task`

```swift
struct DataView: View {
    @State private var data: [Item] = []
    @State private var isLoading = false
    @State private var error: Error?

    var body: some View {
        List(data) { item in
            Text(item.name)
        }
        .overlay {
            if isLoading {
                ProgressView()
            }
        }
        .task {
            isLoading = true
            defer { isLoading = false }

            do {
                data = try await fetchData()
            } catch {
                self.error = error
            }
        }
    }
}
```

See `references/async-patterns.md` for more `.task`/`.refreshable`/background-task worked examples, and `references/sheet-navigation-patterns.md` for a `NavigationPath`-owning `@Observable` coordinator.

## Decision Flowchart

```
Is this value owned by this view?
├─ YES: Is it a simple value type?
│       ├─ YES → @State private var
│       └─ NO (class):
│           ├─ Use @Observable → @State private var (mark class @MainActor)
│           └─ Legacy ObservableObject → @StateObject private var
│
└─ NO (passed from parent):
    ├─ Does child need to MODIFY it?
    │   ├─ YES → @Binding var
    │   └─ NO: Does child need BINDINGS to its properties?
    │       ├─ YES (@Observable) → @Bindable var
    │       └─ NO: Does child react to changes?
    │           ├─ YES → var + .onChange()
    │           └─ NO → let
    │
    └─ Is it a legacy ObservableObject from parent?
        └─ YES → @ObservedObject var (consider migrating to @Observable)
```

## State Privacy Rules

**All view-owned state should be `private`:**

```swift
// Correct - clear what's created vs passed
struct MyView: View {
    // Created by view - private
    @State private var isExpanded = false
    @State private var viewModel = ViewModel()
    @AppStorage("theme") private var theme = "light"
    @Environment(\.colorScheme) private var colorScheme
    
    // Passed from parent - not private
    let title: String
    @Binding var isSelected: Bool
    @Bindable var user: User
    
    var body: some View {
        // ...
    }
}
```

**Why**: This makes dependencies explicit and improves code completion for the generated initializer.

## Avoid Nested ObservableObject

**Note**: This limitation only applies to `ObservableObject`. `@Observable` fully supports nested observed objects.

SwiftUI can't track changes through nested `ObservableObject` properties. Workaround: pass the nested object directly to child views as `@ObservedObject`. With `@Observable`, nesting works automatically.

## Additional Data-Flow Rules

- **`@State` as a non-observable cache.** If a view stores a class instance holding expensive-to-recompute data (e.g. a `CIContext`), it's fine to hold it in `@State` even though it isn't an `@Observable`/`ObservableObject` — this uses `@State` purely to persist the instance across redraws, with no change tracking implied.
- **Avoid `Binding(get:set:)` in view body code.** It's cleaner to use a binding already provided by `@State`/`@Binding`, then attach `.onChange()` for any side effect:
  ```swift
  // Avoid
  TextField("Username", text: Binding(
      get: { model.username },
      set: { model.username = $0; model.save() }
  ))

  // Prefer
  TextField("Username", text: $model.username)
      .onChange(of: model.username) {
          model.save()
      }
  ```
- **Numeric `TextField` input.** Bind to a numeric value (`Int`/`Double`), not a `String`, and use the `format:` initializer: `TextField("Enter your score", value: $score, format: .number)`. Pair with `.keyboardType(.numberPad)` (integers) or `.keyboardType(.decimalPad)` (floating point) — the modifier alone is not sufficient to get numeric-only entry.
- **Prefer `Identifiable` conformance over `id: \.someProperty`** in `ForEach`/`List` where the type is yours to change — see `references/list-patterns.md` for the full identity-stability discussion.
- **SwiftData**: `ModelContext.fetchCount()` with a fetch descriptor is the efficient way to get just a count, but it will *not* live-update unless something else (like `@Query`) triggers a refresh — use carefully. If the project uses SwiftData with CloudKit: never use `@Attribute(.unique)`, model properties must have default values or be optional, and all relationships must be optional. For anything beyond these quick notes, defer to the dedicated `swiftdata-pro` skill.

## Key Principles

1. **Always prefer `@Observable` over `ObservableObject`** for new code
2. **Mark `@Observable` classes with `@MainActor` for thread safety (unless using default actor isolation)`**
3. Use `@State` with `@Observable` classes (not `@StateObject`)
4. Use `@Bindable` for injected `@Observable` objects that need bindings
5. **Always mark `@State` and `@StateObject` as `private`**
6. **Never declare passed values as `@State` or `@StateObject`**
7. With `@Observable`, nested objects work fine; with `ObservableObject`, pass nested objects directly to child views
8. **Always add `@ObservationIgnored` to property wrappers** (e.g., `@AppStorage`, `@SceneStorage`, `@Query`) inside `@Observable` classes — they conflict with the macro's property transformation
