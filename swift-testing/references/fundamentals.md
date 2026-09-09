# Fundamentals

## When to use this reference

Use this file when creating new Swift Testing suites or refactoring test structure before deeper topics like traits, parameterization, or migration.

## Building blocks

- Import `Testing` only in test targets.
- Use `@Test` to declare tests explicitly (global function or type method).
- Any type containing `@Test` methods is automatically treated as a test suite — you do not need to add `@Suite` for that alone. Add `@Suite` explicitly only when you want a display name or to attach suite-level traits, e.g. `@Suite(.tags(.networking))`.
- Group related tests into suites (`struct`, `actor`, or `class`).
- Prefer `struct` suites for value semantics and accidental-state-sharing prevention. Use a class only if you need subclassing or a `deinit`.
- Use nested suites to reflect feature grouping and improve discoverability.
- You do not need to prefix test methods with `test`. `userCanLogOut()` is fine — no need for `testUserCanLogOut()`.
- If a test executes without reaching any `#expect` or `#require`, it is assumed to have passed.

## Core examples

### Global test function

```swift
import Testing
@testable import FoodTruck

@Test("Food truck has a valid default name")
func defaultName() {
 let truck = FoodTruck()
 #expect(truck.name.isEmpty == false)
}
```

### Suite with instance tests

```swift
import Testing
@testable import FoodTruck

@Suite("Menu tests")
struct MenuTests {
 @Test("Returns no duplicates")
 func uniqueItems() {
 let items = Menu.default.items
 #expect(Set(items).count == items.count)
 }
}
```

### Nested suites for feature grouping

```swift
import Testing
@testable import FoodTruck

struct CheckoutTests {
 struct Taxes {
 @Test func taxIsRoundedToTwoDigits() {
 let total = Checkout.total(subtotal: 10.00, taxRate: 0.0825)
 #expect(total == 10.83)
 }
 }

 struct Discounts {
 @Test func promoCodeAppliesFixedAmount() {
 let total = Checkout.total(subtotal: 20.00, discount: .fixed(5))
 #expect(total == 15.00)
 }
 }
}
```

## No more `setUp`/`tearDown`

Don't carry over XCTest's `setUp()`/`tearDown()` pattern. Use `init()` in structs, `init()`/`deinit()` in classes, or test scoping traits for more advanced per-test setup (see `references/new-features.md`).

```swift
struct PlayerTests {
    let sut: Player

    init() {
        sut = Player(name: "Natsuki Subaru")
    }

    @Test func nameIsCorrect() {
        #expect(sut.name == "Natsuki Subaru")
    }
}
```

## Recommended defaults

- Keep tests small and behavior-focused.
- Prefer descriptive names over boilerplate `test...` prefixes.
- Use display names where human-readable output helps triage.
- Keep setup local, or centralize in suite `init` when shared across tests.
- Avoid hidden global mutable state (see `references/writing-better-tests.md`).
- Use `@MainActor` only when code under test requires main-thread isolation.
- Suite initializers can be `async` and/or `throws`, same as test functions.

## Organization guidance

- Group by feature behavior, not by implementation class only.
- Promote shared traits (e.g. tags) to suite level when all tests inherit them.
- Use tags for cross-cutting grouping across files/targets.
- Keep unrelated tests in separate suites to preserve clear ownership.

## Suite constraints to enforce

- If a suite has instance test methods, it must have a callable zero-argument initializer (implicit or explicit, sync/async, throwing or non-throwing). If any properties are added, they must either have default values or be set by a custom zero-arg-callable initializer.
- If initialization requirements cannot be met, convert tests to static/global functions or refactor suite state.
- Suite types (and containing types) must always be available; do not apply `@available` to suite declarations — only to individual test functions. If a whole suite happens to only contain tests written for one OS version, annotate each test individually rather than the suite.

### Zero-argument initializer requirement example

```swift
import Testing

@Suite
struct SessionTests {
 let config: URLSessionConfiguration

 // Valid: callable with zero args due to default value.
 init(config: URLSessionConfiguration = .ephemeral) {
 self.config = config
 }

 @Test func usesEphemeralByDefault() {
 #expect(config == .ephemeral)
 }
}
```

### Invalid availability placement

```swift
import Testing

// Do not do this on suite types:
// @available(iOS 18, *)
@Suite
struct PushTests {
 @available(iOS 18, *)
 @Test func supportsNewPushFormat() {
 #expect(true)
 }
}
```

## Do / Don't

- Do keep each test focused on one behavior.
- Do use display names where they improve failure readability.
- Don't rely on test execution order.
- Don't annotate suites with `@available`; annotate test functions instead.
- Don't add `@Suite` purely out of habit — only when it does something (name or traits).

## Review checklist

- Test target imports `Testing`, app targets do not.
- Suite choice (`struct`/`actor`/`class`) matches setup and teardown needs.
- Instance tests have a callable zero-argument init path.
- Availability is applied to test functions, not suite types.
