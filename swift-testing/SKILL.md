---
name: swift-testing
description: Use whenever writing, reviewing, improving, migrating, or debugging Swift tests. Covers @Test, #expect, #require, traits, tags, parameterized tests, async/await and confirmation-based tests, parallel execution and isolation, and XCTest-to-Swift-Testing migration. Trigger this for ANY Swift test file work -- new tests, test code review, flaky-test debugging, "modernize my XCTest suite," or a request to write/improve/review tests in a Swift or Apple-platform project -- not only when the user says "Swift Testing" by name.
---

# Swift Testing

## Overview

Swift Testing replaces XCTest with a modern macro-based approach: `@Test` instead of `test`-prefixed methods, `#expect`/`#require` instead of `XCTAssert*`, structs instead of `XCTestCase` subclasses, and parallel-by-default execution instead of serial. If you learned XCTest, unlearn it — Swift Testing works differently in ways that matter (state isolation, argument combinatorics, actor behavior).

Use this skill to write, review, migrate, and debug Swift tests with modern Swift Testing APIs. Prioritize readable tests, robust parallel execution, clear diagnostics, and incremental migration from XCTest where needed. When asked to review code, report only genuine problems — do not nitpick or invent issues.

- [Apple Documentation](https://developer.apple.com/documentation/testing)
- [Migration Guide](https://steipete.me/posts/2025/migrating-700-tests-to-swift-testing)

## Agent behavior contract

1. Prefer Swift Testing for all new Swift unit and integration tests, but keep XCTest for UI automation (`XCUIApplication`), performance metrics (`XCTMetric`), and Objective-C-only test code — Swift Testing does not support UI tests.
2. Treat `#expect` as the default assertion; use `#require` only when a later line depends on the value being present/true (precondition, not assertion).
3. Default to parallel-safe guidance. If tests aren't isolated, propose fixing the shared state first rather than reaching for `.serialized`.
4. Prefer traits for behavior and metadata (`.enabled`, `.disabled`, `.timeLimit`, `.bug`, tags) over naming conventions or ad-hoc comments.
5. Recommend parameterized tests when multiple tests share logic and differ only in input values — but watch argument-count combinatorics (see `references/parameterized-testing.md`).
6. Use `@available` on individual test *functions* for OS-gated behavior, never on suite types.
7. Keep migration advice incremental: convert assertions first, then organize suites, then introduce parameterization/traits. Don't rewrite an existing XCTest suite to Swift Testing unless asked.
8. Only import `Testing` in test targets, never in app/library/binary targets.
9. Target Swift 6.2+ and modern Swift concurrency by default, and use a project structure with folder layout that mirrors the code under test (see `references/writing-better-tests.md`).
10. Swift Testing gains new features every Swift release (3-4/year), so your training data is likely stale on recent APIs and Apple's own docs can lag too. Treat the user's installed toolchain as authoritative for what's available; see `references/new-features.md` for capabilities that may postdate your training.

## First 60 seconds (triage template)

- Clarify the goal: new tests, migration, flaky failures, performance, CI filtering, or async waiting.
- Collect minimal facts:
  - Xcode/Swift version and platform targets
  - Whether tests currently use XCTest, Swift Testing, or both
  - Whether failures are deterministic or flaky
  - Whether tests access shared resources (database, files, network, global state)
- Branch quickly:
  - repetitive tests -> parameterized tests
  - noisy or flaky failures -> known-issue handling and test isolation
  - migration questions -> XCTest mapping and coexistence strategy
  - async callback complexity -> continuation/confirmation patterns

## Routing map (read the right reference fast)

- Test building blocks, suite structure, struct-vs-class, zero-arg init -> `references/fundamentals.md`
- `#expect`, `#require`, throw expectations, `Issue.record()`, readable failures -> `references/expectations.md`
- Traits, tags, bug linking, conditions, availability -> `references/traits-and-tags.md`
- Parameterized test design, combinatorics, `zip` pitfalls -> `references/parameterized-testing.md`
- Async/await tests, confirmations, callback bridging, actor isolation, `.serialized` nuances, mocking -> `references/async-testing.md`
- Default parallel execution, random order, suite-level isolation strategy -> `references/parallelization-and-isolation.md`
- Test speed, determinism, and flakiness prevention -> `references/performance-and-best-practices.md`
- Test hygiene: FIRST principles, structuring tests, hidden dependencies, testing SwiftUI view models -> `references/writing-better-tests.md`
- Recent Swift Testing features (raw identifiers, exit tests, attachments, test scopes, `ConditionTrait.evaluate()`) -> `references/new-features.md`
- XCTest coexistence and migration workflow -> `references/migration-from-xctest.md`
- Xcode test navigator/report workflows and diagnostics -> `references/xcode-workflows.md`

If doing partial work (e.g. only a migration pass, or only an async-tests review), load only the relevant reference files instead of all of them.

## Common pitfalls -> next best move

- Repetitive `testFooCaseA/testFooCaseB/...` methods -> replace with one parameterized `@Test(arguments:)` (`references/parameterized-testing.md`).
- Two argument collections without `zip` -> silently becomes a Cartesian product, not pairwise; use `zip`, or better, an array of tuples/dictionary to avoid `zip`'s silent-truncation and enum-reordering fragility (`references/parameterized-testing.md`).
- Failing optional preconditions hidden in later assertions -> `try #require(...)` then assert on the unwrapped value (`references/expectations.md`).
- Overusing `#require` for ordinary assertions -> stops the test at first failure instead of reporting all failures; reserve it for preconditions (`references/expectations.md`).
- `#expect(!isLoggedIn)` -> `!` defeats macro expansion and produces unhelpful failure output; write `#expect(isLoggedIn == false)` instead (`references/expectations.md`).
- "Each test gets a fresh instance, so state can't leak" -> true for instance properties, false for `static`/singleton state; isolate or reset it (`references/parallelization-and-isolation.md`).
- Flaky integration tests on shared database -> isolate dependencies or use in-memory repositories; use `.serialized` only as a transition step (`references/parallelization-and-isolation.md`).
- `.serialized` "should" work on any test -> it only affects a parameterized test's own cases when applied directly to a single `@Test`; applied to a `@Suite` it serializes everything inside that suite (`references/async-testing.md`).
- `.timeLimit(.seconds(10))` -> wrong; the trait only accepts `.minutes(...)` (`references/async-testing.md`).
- Wrapping async work in `Task { }` inside a test, or using a completion closure with `confirmation()` -> defeats the point; use `async` test functions directly, or track the `Task` and `await` it (`references/async-testing.md`).
- `confirmation()` used for general assertions -> it's for verifying callback/event counts, not a substitute for `#expect` (`references/async-testing.md`).
- Disabled tests that silently rot -> prefer `withKnownIssue` (optionally `isIntermittent: true`) over blanket disabling so they keep signaling (`references/expectations.md`).
- Unclear failure output for complex types -> conform to `CustomTestStringConvertible` in the test target only (`references/expectations.md`).
- Test-plan include/exclude by test name -> use tags and tag-based filters instead (`references/traits-and-tags.md`, `references/xcode-workflows.md`).
- Expected value derived from the same expression as the code under test, or `if`/`switch` branching inside a parameterized test body -> both let the test mirror/mask bugs in production logic instead of verifying it independently (`references/parameterized-testing.md`).
- Testing a SwiftUI `View` directly -> flaky and implementation-coupled; test the view model instead (`references/writing-better-tests.md`).
- Hidden dependencies (`URLSession.shared`, ambient `UserDefaults`) baked into production code -> inject them so tests can substitute fakes (`references/writing-better-tests.md`).

## Reviewing or writing test code

When asked to review Swift Testing code, organize findings by file. For each issue:

1. State the file and relevant line(s).
2. Name the rule being violated.
3. Show a brief before/after code fix.

Skip files with no issues. End with a prioritized summary of the most impactful changes to make first.

When asked to write or improve tests, follow the same rules above but make the changes directly instead of returning a findings report.

Example finding:

```
### UserTests.swift

**Line 5: Use struct, not class, for test suites.**

// Before
class UserTests: XCTestCase {
// After
struct UserTests {

**Line 30: Use `#require` for preconditions, not `#expect`.**

// Before
#expect(users.isEmpty == false)
let first = users.first!
// After
let first = try #require(users.first)

### Summary
1. **Fundamentals (high):** Test suite on line 5 should be a struct, not a class inheriting from `XCTestCase`.
2. **Assertions (medium):** Force-unwrap on line 30 should use `#require` to unwrap safely and stop the test early on failure.
```

## Verification checklist

- Each test has a single clear behavior and an expressive display name when needed.
- Prerequisites use `#require` where failure should stop the test; ordinary assertions use `#expect`.
- Repeated logic is parameterized instead of duplicated, with concrete (not derived) expected values.
- Tests are parallel-safe, or intentionally `.serialized` with a stated reason.
- Async code is awaited natively (not `Task { }`-wrapped), and callback APIs are bridged safely.
- Hidden dependencies (networking, `UserDefaults`, time, randomness) are injected, not ambient.
- Migration keeps unsupported XCTest-only scenarios (UI tests, `XCTMetric`) on XCTest.
- Test file/folder structure mirrors the production code it covers.

## References

- `references/fundamentals.md`
- `references/expectations.md`
- `references/traits-and-tags.md`
- `references/parameterized-testing.md`
- `references/async-testing.md`
- `references/parallelization-and-isolation.md`
- `references/performance-and-best-practices.md`
- `references/writing-better-tests.md`
- `references/new-features.md`
- `references/migration-from-xctest.md`
- `references/xcode-workflows.md`
