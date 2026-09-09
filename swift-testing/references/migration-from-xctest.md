# Migration from XCTest

## When to use this reference

Use this file for incremental migration of existing XCTest code to Swift Testing while preserving safety and CI signal.

## Don't rewrite unasked

If the project has existing XCTest tests, do *not* rewrite them to Swift Testing unless requested. Even when migrating, remember XCTest supports UI testing (`XCUIApplication`) and performance metrics (`XCTMetric`), which Swift Testing does not — those scenarios stay on XCTest regardless of how much else migrates.

## Coexistence strategy

- Swift Testing and XCTest can coexist in the same target.
- Migrate incrementally; do not block migration on a full rewrite.
- A single source file can import both `XCTest` and `Testing` during migration.
- Keep XCTest where Swift Testing does not apply: UI automation (`XCUIApplication`), performance APIs (`XCTMetric`), Objective-C-only tests.

```swift
import XCTest
import Testing
```

## Practical migration order

1. Convert assertions to `#expect` / `#require`.
2. Replace `test...`-prefix naming constraints with explicit `@Test`.
3. Reorganize classes into suites (structs) where helpful.
4. Collapse repetitive methods into parameterized tests.
5. Add traits/tags for control and test-plan filtering.

Or, phrased as a conversion checklist when migrating a specific suite:

1. Start by keeping the same broad structure: same type names (just class -> struct), same test methods (drop the `test` prefix, add `@Test`), switching old-style assertions to new-style expectations.
2. Look for places where parameterized tests can cut down on test code or improve coverage.
3. Add `#require` checks at the start of tests for preconditions.
4. Finish by adding traits where appropriate — `.timeLimit()`, `.enabled(if:)`, `.tags()`, etc. — to replace XCTest conventions like conditionally skipping tests.

## Example conversion: class method -> Swift Testing function

```swift
// Before (XCTest)
final class PriceTests: XCTestCase {
 func testDiscountedTotal() {
 XCTAssertEqual(Price.total(subtotal: 20, discount: 5), 15)
 }
}

// After (Swift Testing)
import Testing

@Test func discountedTotal() {
 #expect(Price.total(subtotal: 20, discount: 5) == 15)
}
```

## Assertion mapping table

| XCTest | Swift Testing |
|--------|---------------|
| `XCTAssert(expr)` / `XCTAssertTrue(expr)` | `#expect(expr)` |
| `XCTAssertEqual(a, b)` | `#expect(a == b)` |
| `XCTAssertLessThan(a, b)` | `#expect(a < b)` |
| `XCTAssertNil(a)` | `#expect(a == nil)` |
| `XCTAssertNotNil(a)` | `#expect(a != nil)` |
| `XCTAssertIdentical(a, b)` | `#expect(a === b)` — same object instance |
| `try XCTUnwrap(a)` | `try #require(a)` — also works with any Boolean condition, not just optionals |
| `XCTAssertThrowsError` | `#expect(throws: ErrorType.self) { }` |
| `XCTAssertNoThrow` | `#expect(throws: Never.self) { }` |
| `XCTFail("message")` | `Issue.record("message")` |

...and so on for the rest of the `XCTAssert*` family.

### Table-style quick mappings

```swift
// XCTAssertTrue(isEnabled)
#expect(isEnabled)

// XCTAssertNil(error)
#expect(error == nil)

// XCTAssertThrowsError(try run())
#expect(throws: (any Error).self) { try run() }

// try XCTUnwrap(user)
let user = try #require(user)
```

## Floating-point tolerance

Swift Testing does *not* offer built-in float tolerance for "close enough" comparisons the way some XCTest helpers implied. Bring in Apple's Swift Numerics library and use `isApproximatelyEqual(to:absoluteTolerance:)`:

```swift
#expect(celsius.isApproximatelyEqual(to: 0, absoluteTolerance: 0.000001))
```

**Important:** Unless Swift Numerics is already imported into the project, do not add it as a dependency without first asking the user.

## Suite model differences

- XCTest: class + `XCTestCase`.
- Swift Testing: struct/actor/class suites, explicit attributes, value-semantics-friendly defaults.
- Setup can move from `setUp` patterns to a suite `init`; teardown can move to `deinit` on class/actor suites.
- XCTest synchronous tests default to main-actor behavior; Swift Testing runs tests on arbitrary tasks unless explicitly isolated (e.g. `@MainActor`) — don't assume migrated tests keep XCTest's implicit main-actor behavior.

```swift
import Testing

struct SessionTests {
 let session: Session

 init() {
 self.session = Session(environment: .test)
 }

 @Test func startsDisconnected() {
 #expect(session.isConnected == false)
 }
}
```

## Async migration specifics

- Prefer `await` directly for async APIs.
- Convert completion-handler APIs with `withCheckedContinuation`/`withCheckedThrowingContinuation` (see `references/async-testing.md`).
- Replace `XCTestExpectation` patterns with confirmations when testing asynchronous event streams.

```swift
import Testing

@Test func receivesAtLeastOneEvent() async {
 await confirmation("Receives event", expectedCount: 1...) { confirm in
 confirm()
 }
}
```

## Migration hygiene

- Prefer mechanical, reviewable commits.
- Use editor pattern-replace to accelerate common assertion conversions.
- Avoid mixing XCTest assertions in Swift Testing tests (and vice versa).

## Common pitfalls

- Migrating all files at once instead of phased migration.
- Keeping `continueAfterFailure` patterns instead of targeted `#require`.
- Marking every migrated test `@MainActor` unnecessarily (see suite model differences above — it's tempting because XCTest implied main-actor behavior, but usually wrong).
