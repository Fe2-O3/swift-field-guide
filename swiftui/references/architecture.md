# SwiftUI Architecture

Choosing between Apple's built-in patterns (`@Observable` + State-as-Bridge), MVVM, and TCA (The Composable Architecture), plus common anti-patterns and a code-review checklist.

## Architecture Decision Tree

```
- Small/medium app, Apple's patterns? -> @Observable + State-as-Bridge
- Familiar with MVVM from UIKit? -> MVVM with @Observable ViewModels
- Rigorous testability, large team? -> TCA (Composable Architecture)
- Complex navigation, deep linking? -> Add Coordinator Pattern
```

## Property Wrapper Decision

```
- View owns the model? -> @State
- App-wide model? -> @Environment
- Need bindings to parent's model? -> @Bindable
- Just reading? -> Plain property (no wrapper)
```

## State-as-Bridge Pattern (WWDC 2025)

Async work creates suspension points that break animations if state changes happen inside the `Task` without an explicit animation transaction:

```swift
// WRONG - state changes are not wrapped, animation may not apply predictably
Task { isLoading = true; await work(); isLoading = false }

// CORRECT - synchronous, explicitly-animated state changes bridge into/out of the async gap
withAnimation { isLoading = true }
Task {
    await work()
    withAnimation { isLoading = false }
}
```

## MVVM Structure (concise)

```swift
// Model - domain logic
struct Pet: Identifiable {
    let id: UUID; var name: String
    mutating func giveAward() { hasAward = true }
}

// ViewModel - presentation logic
@Observable
class PetListViewModel {
    private let petStore: PetStore
    var searchText = ""

    var filteredPets: [Pet] {
        petStore.myPets.filter { searchText.isEmpty || $0.name.contains(searchText) }
    }
}

// View - UI only
struct PetListView: View {
    @Bindable var viewModel: PetListViewModel

    var body: some View {
        List(viewModel.filteredPets) { PetRow(pet: $0) }
            .searchable(text: $viewModel.searchText)
    }
}
```

## MVVM Structure — full worked example (dependency injection, loading, errors)

A more complete MVVM shape, showing constructor-injected services, loading state, and error surfacing:

```swift
import Observation

@Observable
@MainActor
final class ArticleListViewModel {
    var articles: [Article] = []
    var isLoading = false
    var errorMessage: String?

    private let articleService: ArticleService

    init(articleService: ArticleService) {
        self.articleService = articleService
    }

    func loadArticles() async {
        isLoading = true
        errorMessage = nil

        do {
            articles = try await articleService.fetchArticles()
        } catch {
            errorMessage = error.localizedDescription
        }

        isLoading = false
    }
}

struct ArticleListView: View {
    @State private var viewModel: ArticleListViewModel

    init(articleService: ArticleService) {
        _viewModel = State(wrappedValue: ArticleListViewModel(articleService: articleService))
    }

    var body: some View {
        List(viewModel.articles) { article in
            ArticleRow(article: article)
        }
        .overlay {
            if viewModel.isLoading {
                ProgressView()
            }
        }
        .alert("Error", isPresented: .constant(viewModel.errorMessage != nil)) {
            Button("OK") { viewModel.errorMessage = nil }
        } message: {
            if let message = viewModel.errorMessage {
                Text(message)
            }
        }
        .task {
            await viewModel.loadArticles()
        }
    }
}
```

**Benefits over `ObservableObject`-based MVVM:** no `@Published` needed, fine-grained observation (only tracks accessed properties), less boilerplate.

## TCA Trade-offs

| Scenario | Choice |
|----------|--------|
| < 10 screens | Apple patterns |
| Testability critical | TCA |
| Large team | TCA for consistency |
| Rapid prototyping | Apple patterns |

For TCA specifics (reducers, `Store`, `Effect`, `TestStore`, navigation), defer to the dedicated `composable-architecture` skill rather than duplicating it here.

## Anti-Patterns

**Logic in view body:**
```swift
// WRONG - formatter created every render
var body: some View {
    let formatter = NumberFormatter()
    Text(formatter.string(from: price)!)
}

// CORRECT - cache in model
class ViewModel {
    private let formatter = NumberFormatter()
    func format(_ price: Decimal) -> String { ... }
}
```

**Wrong property wrapper:**
```swift
// WRONG - @State copies, loses parent changes
struct DetailView: View { @State var item: Item }

// CORRECT
struct DetailView: View { let item: Item }  // or @Bindable
```

**God ViewModel:**
```swift
// WRONG
class AppViewModel { var user; var settings; var posts; ... }

// CORRECT - separate concerns
class UserViewModel { }
class SettingsViewModel { }
```

**Architecture mismatch mid-project** — starting with `@Observable` + State-as-Bridge and then discovering you need TCA partway through is expensive to unwind. Choose architecture upfront based on expected complexity (small app = `@Observable`, complex/large-team/testability-critical = TCA), not by default.

## Code Review Checklist

- [ ] View bodies contain ONLY UI code
- [ ] No formatters (or other expensive objects) created in view body
- [ ] Business logic testable without SwiftUI
- [ ] State changes for animations are synchronous (State-as-Bridge)
