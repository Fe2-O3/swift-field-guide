# Diagnostics

Maps common strict-concurrency compiler errors to an ordered list of likely fixes — try them in order rather than jumping straight to the last resort. For the underlying concepts, follow the links into `sendable.md` / `actors.md`.

## "Sending 'x' risks causing data races"

The compiler found a value crossing an isolation boundary where it could still be accessed from the sending side.

Likely fixes (try in order):

1. **Check whether region-based isolation already handles it.** If the sender demonstrably stops using the value after passing it, the compiler may accept it without changes. Avoid adding `Sendable` prematurely. See `sendable.md#region-based-isolation`.
2. **Mark the parameter `sending`.** This tells the compiler the caller transfers ownership and won't touch the value afterward. See `sendable.md#the-sending-keyword`. (This can be useful, but is not that common.)
3. **Make the type `Sendable`** if it genuinely can be shared safely (value type, immutable class, or internally synchronized).
4. **Check whether `nonisolated(nonsending)` resolves it.** If the function no longer hops executors, the value may not actually cross a boundary. See `threading.md`.
5. **Last resort: `@unchecked Sendable`** only if the type uses manual synchronization (locks) and you've verified correctness. See `sendable.md#unchecked-sendable`.

## "Static property 'x' is not concurrency-safe"

A global or static variable is accessible from multiple isolation domains with no protection. See `sendable.md#global-variables` for the three standard solutions (`@MainActor` annotation, `Sendable` conformance if truly immutable, or `nonisolated(unsafe)` as a last resort) — the only diagnostic-specific nuance is: if the entire module defaults to main-actor isolation, a similar declaration in another target may behave differently purely because of that build setting, not because of a code difference.

## "Capture of 'x' with non-sendable type in a `@Sendable` closure"

A closure that crosses isolation boundaries (e.g., passed to `Task {}`, `Task.detached {}`, or `addTask`) captures a non-Sendable value.

Likely fixes:

1. **Check whether the captured value can be made `Sendable`.** Structs and enums with only `Sendable` stored properties just need the conformance declared. Final classes with immutable (`let`) stored properties can conform too.
2. **Restructure to avoid the capture.** Pass the needed data as a parameter to the task rather than closing over a large non-Sendable object. For example, `let id = object.id; Task { use(id) }`.
3. **Move the work onto the same actor.** If the closure doesn't need to run concurrently, keep it on the caller's actor. See `actors.md#isolation-mindset` in `threading.md`.
4. **Use `sending` on the parameter** if you can transfer ownership cleanly. This is relatively niche.

It's tempting to reach for `@unchecked Sendable`, but rarely a good idea unless the user is *absolutely certain* their code is safe.

## "Main actor-isolated conformance of 'X' to 'Y' cannot be used in nonisolated context"

An isolated conformance (e.g., `extension X: @MainActor Y`) is being used from code that doesn't share that isolation. The compiler prevents this because calling the protocol methods off-actor would be a data race. This is distinct from the `SendableMetatype` diagnostic in `actors.md` — that one is about passing the *metatype* across isolation, this one is about calling the conformance's methods.

Likely fixes:

1. **Move the use site onto the same actor.** If the consuming code can be `@MainActor`, the conformance is usable.
2. **Remove the isolation from the conformance** if the protocol methods don't actually need actor-protected state.

## "Expression is 'async' but is not marked with 'await'"

A call crosses an isolation boundary and requires an async hop. This often surprises when calling actor-isolated methods from outside the actor, or when accessing `@MainActor` state from a non-isolated context.

Likely fix: Add `await`. If the call is in synchronous code that cannot be made async, wrap it in `Task {}` (but see `tasks.md` for when that's appropriate).
