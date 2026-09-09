# Expectations

## When to use this reference

Use this file when writing assertions, migrating from `XCTAssert*`, testing thrown errors, or documenting known failures.

## `#expect` as the default

- Use `#expect` for most assertions.
- Pass natural Swift expressions (`==`, `>`, `.contains`, `.isEmpty`, etc.).
- Rely on captured sub-expression values for rich diagnostics in Xcode.
- Avoid old XCTest assertion families in Swift Testing tests.
- Never use `!` to negate a Boolean inside `#expect`/`#require` — it defeats the macro's expansion and produces unhelpful failure output. `#expect(!isLoggedIn)` is bad; `#expect(isLoggedIn == false)` is good and reports properly on failure.

### Example: expressive assertions

```swift
import Testing

@Test func pricingRules() {
 let subtotal = 25
 let discount = 5
 let total = subtotal - discount

 #expect(total == 20)
 #expect(total > 0)
 #expect([10, 20, 30].contains(total))
}
```

## `#require` for prerequisites

- Use `try #require(...)` when later assertions depend on this condition.
- Treat `#require` as "guard + fail test early." Both `#expect` and `#require` fail the test if the condition is false, but `#require` throws on failure and stops the rest of the test from running — this is the right choice for checking assumptions at the start of a test, since if the assumption is wrong, everything after it is meaningless noise.
- `#require` also unwraps optionals, which is cleaner and safer than force-unwrapping in tests.
- Using `#require` requires adding `throws` to the test method.
- Prefer this pattern over manual optional checks or `!` force-unwraps when failure should halt test flow.

### Example: optional precondition + unwrapped usage

```swift
import Testing

@Test func parsedURLHasHTTPS() throws {
 let value = "https://www.avanderlee.com"
 let url = try #require(URL(string: value), "URL should parse")
 #expect(url.scheme == "https")
}
```

### Example: precondition before the real assertion

```swift
@Test func outstandingTasksStringIsPlural() throws {
    let sut = try createTestUser(projects: 3, itemsPerProject: 10)
    try #require(sut.projects.isEmpty == false)
    let rowTitle = sut.outstandingTasksString
    #expect(rowTitle == "30 items")
}
```

If the `#require` fails, the test stops immediately rather than producing confusing secondary failures. Use `#expect` for the assertions you actually care about, `#require` only for preconditions that must hold for the test to be meaningful.

## Throwing behavior checks

- For success-path calls to throwing functions, call directly and assert the returned value.
- For expected failure, use throw-aware expectations to verify: any throw, a specific error type, or a specific error case/value.
- Always name the specific error rather than a broad `Error.self` — `#expect(throws: Error.self) { ... }` passes for *any* error, which hides regressions where the wrong error is thrown.
- Avoid verbose hand-written `do/catch` unless custom branching is truly needed — but see the `Issue.record()` pattern below when you do need that fine-grained control.

### Example: expected throw and no-throw

```swift
import Testing

enum BrewError: Error, Equatable {
 case missingBeans
}

func brew(_ hasBeans: Bool) throws -> String {
 guard hasBeans else { throw BrewError.missingBeans }
 return "coffee"
}

@Test func expectedThrows() {
 #expect(throws: BrewError.self) {
 try brew(false)
 }
}

@Test func expectedNoThrow() {
 #expect(throws: Never.self) {
 try brew(true)
 }
}
```

```swift
#expect(throws: (any Error).self) { try riskyOperation() }
#expect(throws: NetworkError.self) { try fetch() }
#expect(throws: NetworkError.timeout) { try fetch() }
#expect(throws: Never.self) { try safeOperation() }
```

## `Issue.record()` for fine-grained throw testing

When you need to assert on the *specific* error case and fail explicitly with a custom message if the wrong error is thrown, a `do`/`try`/`catch` block with `Issue.record()` gives more control than `#expect(throws:)`. If no error is thrown, execution falls through `try` and hits `Issue.record()`, failing the test:

```swift
@Test func playingMinecraftThrows() {
    let game = Game(name: "Minecraft")

    do {
        try game.play()
        Issue.record("Expected an error to be thrown.")
    } catch GameError.notPurchased {
        // success
    } catch {
        Issue.record("Wrong error thrown: \(error)")
    }
}
```

`XCTFail("...")` migrates directly to `Issue.record("...")`.

## Known issue handling

- Use `withKnownIssue` for temporary expected failures you still want to compile and run, instead of blanket-disabling the test.
- `withKnownIssue` expects a failure to occur inside its closure, and *fails the test* if no issue is recorded — it's a positive assertion that the known bug is still there, not a way to silence a section.
- Add `isIntermittent: true` for flaky issues you're actively debugging: this flips the semantics so the test passes if no issue is recorded, but marks an expected (non-failing) issue if one is.
- Remove the wrapper once the underlying failure condition is fixed.

### Example: scope only the failing section

```swift
import Testing

@Test func checkoutFlow() {
 #expect(true) // still validated

 withKnownIssue("Checkout backend intermittently returns 503", isIntermittent: true) {
 Issue.record("Known upstream issue")
 }

 #expect(2 + 2 == 4) // rest of test still executes
}
```

## Readability upgrade: `CustomTestStringConvertible`

Conform complex domain types to `CustomTestStringConvertible` for concise, readable test output. Add this conformance in the test target only — never in production code.

Without it, a caught error might print as:

```
Test patchMatchThrows() recorded an issue at ThrowingTests.swift:61:6: Caught error: parentalControlsDisallowed
```

With a retroactive conformance in the test target:

```swift
extension GameError: @retroactive CustomTestStringConvertible {
    public var testDescription: String {
        switch self {
        case .notPurchased:
            "This game has not been purchased."
        case .notInstalled:
            "This game is not currently installed."
        case .parentalControlsDisallowed:
            "This game has been blocked by parental controls."
        }
    }
}
```

```swift
struct Receipt: CustomTestStringConvertible {
 let id: UUID
 let total: Decimal

 var testDescription: String {
 "Receipt(total: \(total))"
 }
}
```

## Writing good verification methods

Verification methods wrap multiple expectations to keep other tests concise. Use `SourceLocation` and the `#_sourceLocation` macro (the leading underscore is required) so failed expectations report the location of the *calling* test rather than a line inside the verification helper:

```swift
func verifyDivision(_ result: (quotient: Int, remainder: Int), expectedQuotient: Int, expectedRemainder: Int, sourceLocation: SourceLocation = #_sourceLocation) {
    #expect(result.quotient == expectedQuotient, sourceLocation: sourceLocation)
    #expect(result.remainder == expectedRemainder, sourceLocation: sourceLocation)
}
```

`#require` also accepts `sourceLocation:` — pass it through on both if a verification method mixes `#require` and `#expect`.

## XCTest mapping quick examples

```swift
// XCTAssertEqual(total, 20)
#expect(total == 20)

// try XCTUnwrap(user)
let user = try #require(user)

// XCTFail("Unreachable")
Issue.record("Unreachable")
```

See `references/migration-from-xctest.md` for the full assertion mapping table.

## Do / Don't

- Do use `#require` when later checks depend on a value.
- Do keep `withKnownIssue` scopes narrow.
- Do name the specific error type/case in `#expect(throws:)` rather than `Error.self`.
- Don't use XCTest assertions in Swift Testing tests.
- Don't hide prerequisite failures inside later optional chaining or force-unwraps.
- Don't negate a Boolean with `!` inside `#expect`/`#require`.
