# Writing Better Tests

## When to use this reference

Use this file for test hygiene and structure questions that aren't about a specific Swift Testing API: how to organize suites and fixtures, how to avoid hidden dependencies, and how to test SwiftUI-adjacent code. Mostly not about specific APIs — more about how to structure tests for maximum flexibility and effectiveness.

## Encourage unit test hygiene: FIRST

Good unit tests fit the acronym FIRST:

- **Fast**: you should be able to run dozens of them every second, if not hundreds or thousands.
- **Isolated**: they should not depend on another test having run, or any external state.
- **Repeatable**: they should always give the same result regardless of how many times or when they're run.
- **Self-verifying**: the test must unambiguously say whether it passed or failed, with no room for interpretation.
- **Timely**: best written before or alongside the production code being tested.

The "timely" part may already be behind you for existing code, but the other four should be firm goals for any test you write or review.

## Test generation heuristics

For a given function, aim to generate:

- Happy path tests
- Boundary tests
- Invalid input tests
- Concurrency tests, if appropriate

## Testing SwiftUI views

Never test views directly — they use `@State` and are likely to behave unpredictably in a test environment.

Instead, test view models or similar business-logic types. This might mean suggesting the user extract business logic into a more testable mechanism — offer that as a *suggestion*, not something to apply immediately/unprompted.

If the project uses `@Observable` view models, they're directly testable without needing a protocol wrapper — just create an instance and test its properties and methods.

## Structuring tests

Prefer to organize test types in a pattern that mirrors the production code. For example, if production code has a folder called `Extensions` containing `URLSession-Decodable.swift`, the test target should have a matching `Extensions` folder containing `URLSession-Decodable.swift`, testing the contents of that one production file.

**If writing new tests, follow this rule. If working with existing tests that don't already follow it, do not apply it without permission from the user** — retrofitting file layout onto an established test suite is a bigger, separate change.

- Strongly prefer organizing related tests into test suites, ideally following this file/folder structure.
- Put test fixtures in a dedicated file. A handful of fixtures can live in a simple `Fixtures` folder; if there are many and they vary across tests, use multiple `Fixtures` folders placed alongside whatever tests they support.
- Use tags to mark up different kinds of work — see `references/traits-and-tags.md` for suggested categories (at minimum, tag networking tests).
- Add user-facing messages to `#expect`/`#require` when they provide value (not always necessary, but usually is).
- Convert repetitive tests into parameterized tests where it makes sense (`references/parameterized-testing.md`).
- Generally test only one behavior per unit test, though multiple `#expect` lines may be used if needed.

## Expose hidden dependencies

Strongly prefer to avoid hidden dependencies in the production code under test. In Swift apps this is commonly `UserDefaults` or `URLSession`.

For example, this is bad because it has a hidden dependency on `URLSession`:

```swift
struct News {
    var url: URL
    var stories = ""

    mutating func fetch() async throws {
        let (data, _) = try await URLSession.shared.data(from: url)
        stories = String(decoding: data, as: UTF8.self)
    }
}
```

A first step is to inject the `URLSession`, keeping a default so call sites don't need to change:

```swift
func fetch(using session: URLSession = .shared) async throws {
    let (data, _) = try await session.data(from: url)
    stories = String(decoding: data, as: UTF8.self)
}
```

Even better is to wrap `URLSession` in a protocol, requiring only the methods actually used in production code:

```swift
protocol URLSessionProtocol {
    func data(from url: URL) async throws -> (Data, URLResponse)
}

extension URLSession: URLSessionProtocol { }
```

```swift
func fetch(using session: any URLSessionProtocol = URLSession.shared) async throws {
    let (data, _) = try await session.data(from: url)
    stories = String(decoding: data, as: UTF8.self)
}
```

This lets you create a mock `URLSession` for tests, removing live networking entirely, without changing how the method is called in production code. See `references/async-testing.md` for a full mock example.

With `UserDefaults`, the problem is that ambient shared state set elsewhere can make tests fail unpredictably. Switch to dependency injection with a sensible default (whatever the project already used), then pass in a scoped, disposable instance in tests:

```swift
let suite = "suite-\(UUID().uuidString)"
let userDefaults = UserDefaults(suiteName: suite)
defer { userDefaults?.removePersistentDomain(forName: suite) }
```

That creates a local `UserDefaults` instance in the test and ensures it's fully deleted before the test completes.

The same concept applies more broadly: control time, randomness, and any other ambient dependency so meaningful, repeatable tests can be written.

## Do / Don't

- Do mirror production folder/file structure for new test suites.
- Do inject ambient dependencies (`URLSession`, `UserDefaults`, clocks, RNGs) rather than reaching for them statically.
- Do suggest — don't force — extracting view logic out of SwiftUI views for testability.
- Don't test SwiftUI `View` types directly.
- Don't retrofit file-structure conventions onto an existing test suite without asking first.
