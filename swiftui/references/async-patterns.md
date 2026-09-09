# Async Operation Patterns

## `.task(id:)` — Restart on Value Change

`.task { }` alone starts once and cancels automatically on disappear. `.task(id:)` additionally **cancels and restarts** the work whenever the identity value changes — useful for "reload when the selected filter/id changes" without hand-rolled cancellation:

```swift
struct UserListView: View {
    @State private var users: [User] = []
    @State private var selectedFilter: Filter = .all

    var body: some View {
        List(users) { user in
            UserRow(user: user)
        }
        .task {
            await loadUsers()
        }
        .task(id: selectedFilter) {
            // Cancelled and restarted when selectedFilter changes
            await loadUsers(filter: selectedFilter)
        }
    }

    func loadUsers(filter: Filter = .all) async {
        users = (try? await fetchUsers(filter: filter)) ?? []
    }
}
```

Never use `.onAppear { Task { ... } }` as a substitute — it doesn't get SwiftUI's automatic cancellation on disappear, so a slow request can complete after the view is gone and write to state nobody's reading anymore.

**`.task` cancellation doesn't propagate to nested `Task { }` blocks.** SwiftUI cancels the top-level task attached via `.task` when the view disappears, but any `Task { }` you spawn *inside* that closure is a separate, uncancelled unit of work unless you explicitly check `Task.isCancelled` or thread cancellation through (e.g. via `withTaskCancellationHandler` or by storing and cancelling a task handle, as in the debounced-search pattern in `references/performance-patterns.md`). Complex async flows with nested tasks need explicit cancellation tracking to avoid zombie tasks that outlive the view.

## Task Modifier

**Use for:** Loading data when view appears

```swift
struct ArticleDetailView: View {
    let articleId: String
    @State private var article: Article?
    @State private var isLoading = true

    var body: some View {
        Group {
            if let article {
                ArticleContent(article: article)
            } else if isLoading {
                ProgressView()
            } else {
                ContentUnavailableView("Article Not Found", systemImage: "doc.text")
            }
        }
        .task {
            await loadArticle()
        }
    }

    private func loadArticle() async {
        isLoading = true
        defer { isLoading = false }

        do {
            article = try await articleService.fetchArticle(id: articleId)
        } catch {
            print("Error loading article: \(error)")
        }
    }
}
```

## Refreshable Content

**Use for:** Pull-to-refresh lists

```swift
struct ArticleListView: View {
    @State private var articles: [Article] = []

    var body: some View {
        List(articles) { article in
            ArticleRow(article: article)
        }
        .refreshable {
            await refreshArticles()
        }
    }

    private func refreshArticles() async {
        do {
            articles = try await articleService.fetchArticles()
        } catch {
            print("Error refreshing: \(error)")
        }
    }
}
```

## Background Tasks

**Use for:** Non-blocking async operations

```swift
struct ArticleDetailView: View {
    let article: Article
    @State private var isSaved = false

    var body: some View {
        ArticleContent(article: article)
            .toolbar {
                Button(isSaved ? "Saved" : "Save") {
                    Task {
                        await saveArticle()
                    }
                }
            }
    }

    private func saveArticle() async {
        do {
            try await articleService.saveArticle(article)
            isSaved = true
        } catch {
            print("Error saving: \(error)")
        }
    }
}
```
