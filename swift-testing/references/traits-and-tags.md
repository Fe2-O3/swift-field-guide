# Traits and Tags

## When to use this reference

Use this file when controlling test execution behavior, linking bug context, and organizing large test suites for targeted runs and CI filtering.

## Trait categories

- **Informational**: display names, bug links, tags.
- **Conditional**: `.enabled(if:)`, `.disabled(...)`, availability attributes.
- **Behavioral**: `.timeLimit(...)`, `.serialized`.

## Basic trait examples

```swift
import Testing

@Test("Uploads complete quickly", .timeLimit(.minutes(1)))
func uploadWithinTimeLimit() async throws {
 #expect(true)
}

@Test(.disabled("Flaky on CI while investigating issue"), .bug("https://example.com/issues/12"))
func temporaryDisabledTest() {
 #expect(true)
}
```

Note: `.timeLimit` only accepts `.minutes(...)` — there is no `.seconds()` overload. See `references/async-testing.md` for time-limit inheritance rules between suite and test.

## Conditions and disabling

- Use `.enabled(if:)` or `.disabled(if:)` for runtime-evaluated environments.
- Use `.disabled("reason")` instead of commenting tests out.
- Include actionable reason text in disabled traits for CI/test reports.
- Add `.bug(...)` to link issue trackers and aid future cleanup.

### Runtime condition example

```swift
import Testing

enum Runtime {
 static let isCI = ProcessInfo.processInfo.environment["CI"] == "true"
}

@Test(.enabled(if: Runtime.isCI))
func ciOnlySmokeTest() {
 #expect(true)
}
```

## Availability

- Use `@available` on individual tests when their behavior is entirely OS-gated. Swift Testing supports `@available` on tests but *not* on suites — if a whole suite happens to only contain iOS-26-only tests, annotate each test rather than the suite.
- Prefer `@available` over inline runtime `#available` checks inside the test body for clearer reporting semantics.

```swift
import Testing

@available(iOS 18, *)
@Test func modernPushPayload() {
 #expect(true)
}
```

## Tracking bug fixes

When writing a test for a specific bug, attach the `.bug` trait with the bug ID or URL — this gives future readers context if the bug resurfaces.

```swift
@Test("Headings should always be italic", .bug(id: 182))
```

```swift
@Test("Headings should always be italic", .bug("https://github.com/you/repo/issues/182"))
```

## Tags

- Declare custom tags and apply to tests/suites for cross-suite grouping.
- Use tags for test-plan include/exclude, navigator filtering, and failure analytics.
- Treat tags as cross-cutting metadata, not a replacement for suite structure.
- Use meaningful domain labels (e.g. `networking`, `regression`) over vague terms.
- At minimum, tag network-related tests (even mocked ones) with something like `.networking`. Other useful categories: `.slow` for unexpectedly slow tests, `.edgeCase` for tests needing extra care, `.smoke` for smoke tests.

### Defining and applying tags

```swift
import Testing

extension Tag {
 @Tag static var networking: Self
 @Tag static var regression: Self
}

@Suite(.tags(.networking))
struct APITests {
 @Test func fetchUser() async throws {
 #expect(true)
 }
}

struct CheckoutTests {
 @Test(.tags(.regression))
 func orderTotal() {
 #expect(3 * 3 == 9)
 }
}
```

## Inheritance and scope

- Traits and tags on suites cascade to contained tests.
- Apply at suite level when broadly true; apply per test when specific.
- Keep trait intent explicit to avoid accidental broad behavior changes.

## Do / Don't

- Do put shared tags at suite level for consistency.
- Do attach bug links for temporary disables or known failures.
- Don't use tags as a replacement for meaningful suite grouping.
- Don't overuse `.serialized` as a blanket reliability fix (see `references/parallelization-and-isolation.md`).

## Review checklist

- Every disabled test has a reason (and ideally a bug link).
- Tags reflect domain concerns and are reused consistently.
- Availability and condition traits are applied to the smallest correct scope (tests, not suites).
