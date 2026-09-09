# Async Testing and Waiting

## When to use this reference

Use this file when tests involve async/await functions, completion handlers, streams/events, timing-related flakiness, actor isolation, or `.serialized` behavior.

## Preferred approach

- Use async test functions and `await` naturally.
- Keep async test code close to production async patterns.
- Prefer structured concurrency patterns over ad-hoc synchronization.
- Prefer confirmations for async event-style tests that are not naturally awaitable.
- Do not wrap async operations in `Task { }` inside a test — that defeats the purpose of the test function already being `async`. Use `async`/`await` directly in the test signature: `@Test func testAsync() async throws { }`.

### Async function test example

```swift
import Testing

struct APIClient {
 func fetchName() async throws -> String { "Antoine" }
}

@Test func fetchNameReturnsValue() async throws {
 let client = APIClient()
 let value = try await client.fetchName()
 #expect(value == "Antoine")
}
```

## Callback bridging

For completion-handler APIs without async overloads, bridge with `withCheckedContinuation` / `withCheckedThrowingContinuation`. Keep continuation wrappers minimal and test-focused.

```swift
import Testing

func legacyLoad(_ completion: @escaping (Result<Int, Error>) -> Void) {
 completion(.success(42))
}

@Test func legacyAPI() async throws {
 let value = try await withCheckedThrowingContinuation { continuation in
 legacyLoad { result in
 continuation.resume(with: result)
 }
 }
 #expect(value == 42)
}
```

## Testing pre-concurrency code

If the project contains older concurrency code that relies on callback functions (as opposed to modern `async`/`await`), do not attempt to modernize the *production* code without permission — instead, write tests that wrap the existing callback-based code with `withCheckedContinuation`. The test must wait fully for the completion handler to be called before asserting on its result.

```swift
class ViewModel {
    func loadReadings(completion: @Sendable @escaping ([Double]) -> Void) {
        let url = URL(string: "https://hws.dev/readings.json")!

        URLSession.shared.dataTask(with: url) { data, response, error in
            if let data {
                if let numbers = try? JSONDecoder().decode([Double].self, from: data) {
                    completion(numbers)
                    return
                }
            }

            completion([])
        }.resume()
    }
}
```

```swift
@Test("Loading view model readings")
func loadReadings() async {
    let viewModel = ViewModel()

    await withCheckedContinuation { continuation in
        viewModel.loadReadings { readings in
            #expect(readings.count >= 10, "At least 10 readings must be returned.")
            continuation.resume()
        }
    }
}
```

## Confirmations for asynchronous events

Use confirmations when validating event delivery/count semantics that do not map cleanly to a direct `await`. Set expected counts explicitly (exact count for strict validation, or a lower-bounded range for at-least semantics). Keep confirmation scope small and ensure confirmations happen before the confirmation block returns. `confirmation()` verifies callback/event counts — it is not a substitute for `#expect` assertions.

```swift
import Testing

@Test func eventIsPublishedTwice() async {
 await confirmation("Publishes two events", expectedCount: 2) { confirm in
 confirm()
 confirm()
 }
}
```

### Range-based confirmation counts (Swift 6.1+)

`confirmation(expectedCount:)` also accepts a range, not just a fixed value. Given an async sequence like a `NewsLoader` that yields feeds one at a time:

```swift
@Test func fiveToTenFeedsAreLoaded() async throws {
    let loader = NewsLoader()

    await confirmation(expectedCount: 5...10) { confirm in
        for await _ in loader {
            confirm()
        }
    }
}
```

That fails if `confirm()` is called fewer than 5 or more than 10 times. Partial ranges work too, e.g. `expectedCount: 5...` for "at least five." Ranges without a lower bound (`...10`) are explicitly disallowed, since it's ambiguous whether that means "up to 10" counting from 1 or from 0. `confirmation(expectedCount: 0)` is also valid, and means "ensure this event never happens."

### Confirming work that runs inside a `Task`

When using `confirmation(expectedCount:)` to check that an async function executed a certain number of times, any tested code must have *finished executing fully* by the time the `confirmation()` closure finishes. Using a completion closure inside an untracked `Task` will make the test fail (or worse, flake), because `confirmation()` doesn't know to wait for it:

```swift
// ❌ No way to monitor completion — confirmation() finishes before the Task does.
struct Worker {
    func run(_ work: @escaping () -> Void) -> Task<Void, Never> {
        Task {
            work()
        }
    }
}
```

Fix it one of two ways. Either drop the internal `Task` and make the method itself `async`:

```swift
struct Worker {
    func run(_ work: @escaping () -> Void) async {
        work()
    }
}

@Test
func workerRunsThreeTimes() async {
    let worker = Worker()

    await confirmation(expectedCount: 3) { confirm in
        for _ in 0..<3 {
            await worker.run { /* your work here */ }
            confirm()
        }
    }
}
```

Or, if the production code can't change, return the internal `Task` so the test can track it:

```swift
@Test
func workerRunsThreeTimes() async {
    let worker = Worker()

    await confirmation(expectedCount: 3) { confirm in
        for _ in 0..<3 {
            let task = worker.run { /* simulated work */ }
            await task.value
            confirm()
        }
    }
}
```

## Event handlers and multi-fire callbacks

- Avoid unsafe mutable shared counters from callback closures in strict concurrency mode.
- Use isolation-safe patterns (actor state, `AsyncSequence` wrappers, or thread-safe containers).
- Verify callback counts and ordering explicitly when behavior depends on it.

```swift
import Testing

actor EventCounter {
 private(set) var count = 0
 func increment() { count += 1 }
}

@Test func countEventsSafely() async {
 let counter = EventCounter()
 await counter.increment()
 await counter.increment()
 #expect(await counter.count == 2)
}
```

## Avoid legacy waiting anti-patterns

- Do not return from a test before async callback work completes.
- Avoid sleeping/time-based waits as primary synchronization (`try await Task.sleep(...)` followed by an `#expect` is a flaky anti-pattern).
- Replace brittle waiting with awaitable conditions and deterministic synchronization points (continuations, confirmations, or tracked `Task`s as above).

## Time limits

Time limits are set with the `.timeLimit(...)` trait, specified in `.minutes(...)` — **not** `.seconds()`. This is a common wrong assumption: there is no `.seconds()` overload for `.timeLimit`.

```swift
@Test("Loading view model names", .timeLimit(.minutes(1)))
func loadNames() async {
    let viewModel = ViewModel()
    await viewModel.loadNames()
    #expect(viewModel.names.isEmpty == false, "Names should be full of values.")
}
```

If a suite-level time limit and a test-level time limit both apply, the shorter of the two wins.

## Actor isolation in tests

Isolate a test to a global actor (e.g. `@MainActor`) only when the code under test truly requires it. Keep non-UI tests off the main actor to preserve realistic concurrency behavior and parallelization.

Mark an individual test:

```swift
@MainActor
@Test("Loading view model names")
func loadNames() async {
    // test code here
}
```

Mark a whole suite:

```swift
@MainActor
struct DataHandlingTests {
    @Test("Loading view model names")
    func loadNames() async {
        // test code here
    }
}
```

`confirmation()` and `withKnownIssue()` can also specify an isolation actor for just that closure, letting the rest of the test run elsewhere (main actor or a custom actor):

```swift
@Test("Loading view model names")
func loadNames() async {
    await withKnownIssue("Names can sometimes come back with too few values", isolation: MainActor.shared) {
        // test code here
    }
}
```

Also check whether the test target has default actor isolation enabled project-wide — that can force all tests onto a specific actor without it being obvious from the test code itself.

## `.serialized` — what it actually does

`.serialized` runs test cases one at a time instead of in parallel, but its effect depends on where you put it:

- Applied directly to a single, non-parameterized `@Test`, it has **no effect** — there's only one case to run, so there's nothing to serialize. `.serialized` only changes anything for a parameterized test's own cases (it forces those cases to run one after another instead of in parallel).
- Applied to a `@Suite`, it serializes **all** tests and sub-suites inside that suite relative to each other — this is the common "make this suite single-threaded" use case, and it's independent of whether the individual tests are parameterized.

Agents (and people) very often assume `.serialized` works uniformly on any test; it doesn't. See `references/parallelization-and-isolation.md` for when to reach for it (as a transitional tool, not a default).

## Mocking networking

Unit tests should never do live networking — it's too slow and flaky. Mock the networking layer via a protocol:

```swift
protocol URLSessionProtocol {
    func data(from url: URL) async throws -> (Data, URLResponse)
}

extension URLSession: URLSessionProtocol { }
```

```swift
class URLSessionMock: URLSessionProtocol {
    var testData: Data?
    var testError: (any Error)?

    func data(from url: URL) async throws -> (Data, URLResponse) {
        if let testError {
            throw testError
        } else {
            (testData ?? Data(), URLResponse())
        }
    }
}
```

```swift
@Test func newsStoriesAreFetched() async throws {
    let url = URL(string: "https://www.apple.com/newsroom/rss-feed.rss")!
    var news = News(url: url)
    let session = URLSessionMock()
    session.testData = Data("Hello, world!".utf8)
    try await news.fetch(using: session)
    #expect(news.stories == "Hello, world!")
}
```

See `references/writing-better-tests.md` for the dependency-injection changes to production code that make this possible.
