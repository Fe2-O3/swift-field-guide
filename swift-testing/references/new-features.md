# New Features

## When to use this reference

This document covers recent Swift and Swift Testing features — the kind of thing an LLM's training data is likely to have limited or no coverage of, or where Apple's own documentation may lag the shipped toolchain. Follow the instructions here carefully rather than guessing from older patterns; they're accurate for the version noted, not a hallucination risk.

Treat the user's installed toolchain as the source of truth for what's actually available before relying on a feature below.

## Raw identifiers

**Requires Swift 6.2 or later.**

If the user prefers, use raw identifiers for test names: function names written as natural strings surrounded by backticks, so names read as prose instead of camelCase plus a separate string description.

Instead of:

```swift
@Test("Strip HTML tags from string")
func stripHTMLTagsFromString() {
    // test code
}
```

You can write:

```swift
@Test
func `Strip HTML tags from string`() {
    // test code
}
```

You can put operators such as `+` and `-` into a test method name, but only if they aren't the *only* thing in there.

Raw identifiers combine with parameterized tests too:

```swift
@Test(arguments: [
    (32, 0), (212, 100), (-40, -40),
])
func `Ensure Fahrenheit to Celsius conversion is correct`(values: (input: Double, output: Double)) {
    // test code here
}
```

**Important:** Many users won't know this feature exists, and some will find the style surprising or unwelcome. You can *suggest* raw identifiers to remove duplication, but don't adopt them by surprise unless the project already uses this style.

## Test scoping traits

**Requires Swift 6.1 or later.**

Test scoping traits provide concurrency-safe access to shared test configuration, so each test runs with precise values in place without risking shared mutable state. A common pattern combines them with `@TaskLocal`.

Given production code using a `@TaskLocal` property:

```swift
struct Player {
    var name: String
    var friends = [Player]()

    @TaskLocal static var current = Player(name: "Anonymous")
}

func createWelcomeScreen() -> String {
    var message = "Welcome, \(Player.current.name)!\n"
    message += "Friends online: \(Player.current.friends.count)"
    return message
}
```

Create a test scope by conforming to `TestTrait` and `TestScoping`, implementing `provideScope()` to set up the task local and call `function()`:

```swift
struct DefaultPlayerTrait: TestTrait, TestScoping {
    func provideScope(
        for test: Test,
        testCase: Test.Case?,
        performing function: () async throws -> Void
    ) async throws {
        let player = Player(name: "Natsuki Subaru")

        try await Player.$current.withValue(player) {
            try await function()
        }
    }
}
```

Add a `Trait` extension so the custom trait fits alongside built-in ones:

```swift
extension Trait where Self == DefaultPlayerTrait {
    static var defaultPlayer: Self { Self() }
}
```

Then apply it:

```swift
@Test(.defaultPlayer) func welcomeScreenShowsName() {
    let result = createWelcomeScreen()
    #expect(result.contains("Natsuki Subaru"))
}
```

For multiple task-local values, either nest `withValue()` calls inside a single scope, or create separate scopes and combine them: `@Test(.firstScope, .secondScope, .thirdScope)` — scopes apply in listed order, so later scopes can overwrite values from earlier ones.

Test scopes complement `init()`/`deinit()` — use scopes to opt individual tests or whole suites into configuration as needed.

## Exit tests

**Requires Swift 6.2 or later.**

Swift Testing can test code that results in a critical failure that terminates the process, including deliberate `precondition()`/`fatalError()` calls — this was not possible in XCTest without hacks.

```swift
struct Dice {
    func roll(sides: Int) -> Int {
        precondition(sides > 0)
        return Int.random(in: 1...sides)
    }
}
```

Use `#expect(processExitsWith:)` to catch the critical failure and assert it happened, rather than letting it crash the test run:

```swift
@Test func invalidDiceRollsFail() async throws {
    await #expect(processExitsWith: .failure) {
        let dice = Dice()
        let _ = dice.roll(sides: 0)
    }
}
```

**Important:** This must be executed with `await` — behind the scenes Swift Testing starts a dedicated process for the test, then suspends until that process completes and can be evaluated.

## Attachments

**Requires Swift 6.2 or later.**

Swift Testing can attach debug logs or generated data files to a failing test.

```swift
import Foundation
import Testing

struct Character: Attachable, Codable {
    var id = UUID()
    var name: String
}
```

Because `Character` imports `Foundation` and conforms to `Codable`, Swift Testing can encode instances to attach to tests.

```swift
func makeCharacter() -> Character {
    Character(name: "Ram")
}
```

```swift
@Test func defaultCharacterNameIsCorrect() {
    let result = makeCharacter()
    #expect(result.name == "Rem")

    Attachment.record(result, named: "Character")
}
```

That test fails (the name is wrong on purpose in this example), and Swift Testing surfaces the attachment alongside the failure.

Out of the box, Swift Testing supports attaching `String`, `Data`, and anything `Encodable`. Unless the user has Swift 6.3 available, it does *not* support attaching images. Unlike the XCTest equivalent, Swift Testing's attachments do not support lifetime controls.

## Evaluating `ConditionTrait` outside a test

**Requires Swift 6.2 or later.**

`ConditionTrait` exposes an `evaluate()` method, so non-test code can check the same condition a `@Test` would use:

```swift
struct TestManager {
    static let inSmokeTestMode = true
}

@Test(.disabled(if: TestManager.inSmokeTestMode))
func runLongComplexTest() {
    // test code here
}
```

```swift
func checkForSmokeTest() async throws {
    let trait = ConditionTrait.disabled(if: TestManager.inSmokeTestMode)

    if try await trait.evaluate() {
        print("We're in smoke test mode")
    } else {
        print("Run all tests.")
    }
}
```

## `#expect(throws:)` now returns the error

**Requires Swift 6.1 or later.**

`#expect(_:sourceLocation:performing:throws:)` and `#require(_:sourceLocation:performing:throws:)` — the trailing-closure-pair versions where a second closure validated the caught error — are deprecated.

`#expect(throws:)` and `#require(throws:)` now return the error of the type being checked for, letting you run the expectation and further error validation as separate steps:

```swift
enum GameError: Error {
    case disallowedTime
}

func playGame(at time: Int) throws(GameError) {
    if time < 9 || time > 20 {
        throw GameError.disallowedTime
    } else {
        print("Enjoy!")
    }
}
```

Old, deprecated style:

```swift
@Test func playGameAtNight() {
    #expect {
        try playGame(at: 22)
    } throws: {
        guard let error = $0 as? GameError else { return false }
        return error == .disallowedTime
    }
}
```

New style:

```swift
@Test func playGameAtNight() {
    // `error` will now be a GameError
    let error = #expect(throws: GameError.self) {
        try playGame(at: 22)
    }

    // perform additional validation here
    #expect(error == .disallowedTime)
}
```

## Range-based confirmation counts

**Requires Swift 6.1 or later.** See `references/async-testing.md` for the full write-up (range-based `expectedCount:`, partial ranges, and the disallowed-lower-bound-only case) — it lives there alongside the rest of the confirmation patterns rather than being duplicated here.
