# Bug Patterns

Real concurrency failure modes that LLMs (and humans) produce frequently, for fast review scanning. Entries with a deep-dive elsewhere link out instead of repeating the full explanation; entries below are the ones not covered in depth anywhere else in this skill.

## Continuation resumed zero times

**Failure:** A `withCheckedThrowingContinuation` callback never fires (object deallocated, network timeout with no callback, early return before registering the handler, etc). The caller hangs forever.

**Fix:** Audit every code path to confirm the continuation is resumed. If the underlying API can silently drop the callback, add a timeout or restructure so the caller isn't left waiting. Always use `withCheckedThrowingContinuation` (not the unsafe variant) so that missed resumes are easier to diagnose. See `migration.md#continuations-and-delegate-bridging`.

## Continuation resumed twice

**Failure:** Two callbacks (e.g., a success handler and a cancellation handler) both resume the same continuation. `CheckedContinuation` traps at runtime; `UnsafeContinuation` causes undefined behavior.

**Fix:** Restructure the callback wiring so only one path can reach the continuation. If that isn't possible, guard with a `Bool` flag or use an `actor` to serialize access. Always default to `CheckedContinuation` so double resumes surface immediately during development and testing.

## Swallowed errors in Task closures

**Failure:** `Task { try await riskyWork() }` — if `riskyWork` throws, the error is silently lost. The user sees nothing; the operation just doesn't happen.

**Fix:** Handle the error inside the closure — show an alert, log to a visible surface, or propagate via a `@State` error property.

```swift
Task {
    do {
        try await riskyWork()
    } catch {
        self.errorMessage = error.localizedDescription
    }
}
```

## Blocking the main actor with synchronous work

**Failure:** CPU-intensive work runs on `@MainActor` (or inside `Task {}` called from `@MainActor`), causing UI freezes. This is more likely under Swift 6.2 defaults because `nonisolated` async functions now stay on the caller's executor by default (see `threading.md`).

**Fix:** Move the expensive work into an explicitly offloaded function using `@concurrent`, or use `Task.detached` as a last resort.

## Ignoring `CancellationError` in catch blocks

**Failure:** A `catch` block retries or shows an error alert for `CancellationError`, which is a normal lifecycle event (e.g., user navigated away, view disappeared).

**Fix:** Check for cancellation before handling other errors:

```swift
do {
    try await loadData()
} catch is CancellationError {
    // Normal — view disappeared or task was cancelled. Do nothing.
} catch {
    self.errorMessage = error.localizedDescription
}
```

## Also watch for (covered in depth elsewhere)

These are common enough to belong on a review checklist, but the full explanation lives in the linked reference:

- **Actor reentrancy** (check-then-act across an `await`) — `actors.md#actor-reentrancy`.
- **Unstructured tasks fired in a loop** instead of a task group — `tasks.md#task-groups`.
- **Unbounded `AsyncStream` buffers** under a fast producer / slow consumer — `async-sequences.md#buffer-policies`.
- **`@unchecked Sendable` hiding a real race** (no actual synchronization behind it) — `sendable.md#unchecked-sendable`.
