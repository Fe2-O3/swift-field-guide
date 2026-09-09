# Core Data — Overview & Triage

Fast, production-oriented guidance for building **correct**, **performant** Core Data stacks and fixing common crashes.

This file is the entry point for the `core-data/` reference set (carried over verbatim from the standalone `core-data-expert` skill). Start here, then jump to the specific file below.

## Agent behavior contract (follow these rules)

1. Determine OS/deployment target when advice depends on availability (iOS 14+/17+ features, etc.).
2. Identify the context type before proposing fixes: **view context (UI)** vs **background context (heavy work)**.
3. Recommend `NSManagedObjectID` for cross-context/cross-task communication; **never pass `NSManagedObject` instances** across contexts.
4. Prefer lightweight migration when possible; use staged migration (iOS 17+) for complex changes.
5. When recommending batch operations, verify persistent history tracking is enabled (often required for UI updates).
6. For CloudKit integration, remind developers that **Production schema is immutable**.
7. Reference WWDC/external resources sparingly; prefer this skill's `references/core-data/` files.

## First 60 seconds (triage template)

- **Clarify the goal**: setup, bugfix, migration, performance, CloudKit?
- **Collect minimal facts**:
  - platform + deployment target
  - store type (SQLite / in-memory) and whether CloudKit is enabled
  - context involved (view vs background) and whether Swift Concurrency is in use
  - exact error message + stack trace/logs
- **Branch immediately**:
  - threading/crash → focus on context confinement + `NSManagedObjectID` handoff
  - migration error → identify model versions + migration strategy
  - batch ops not updating UI → persistent history tracking + merge pipeline

## Routing map (pick the right reference fast)

- **Stack setup / merge policies / contexts** → `stack-setup.md`
- **Saving patterns** → `saving.md`
- **Fetch requests / list updates / aggregates** → `fetch-requests.md`
- **Traditional threading (perform/performAndWait, object IDs)** → `threading.md`
- **Swift Concurrency (async/await, actors, Sendable, DAOs)** → `concurrency.md`
- **Batch insert/delete/update** → `batch-operations.md`
- **Persistent history tracking + "batch ops not updating UI"** → `persistent-history.md`
- **Model configuration (constraints, validation, derived/composite, transformables)** → `model-configuration.md`
- **Schema migration (lightweight/staged/deferred)** → `migration.md`
- **CloudKit integration & debugging** → `cloudkit-integration.md`
- **Performance profiling & memory** → `performance.md`
- **Testing patterns** → `testing.md`
- **Terminology** → `glossary.md`
- **Project discovery checklist** → `project-audit.md`
- **Full navigation index** → `_index.md`

## Common errors → next best move

- **"Failed to find a unique match for an NSEntityDescription"** → `testing.md` (shared `NSManagedObjectModel`)
- **`NSPersistentStoreIncompatibleVersionHashError`** → `migration.md` (versioning + migration)
- **Cross-context/threading exceptions** (e.g. delete/update from wrong context) → `threading.md` and/or `concurrency.md` (use `NSManagedObjectID`)
- **Sendable / actor-isolation warnings around Core Data** → `concurrency.md` (don't "paper over" with `@unchecked Sendable`)
- **`NSMergeConflict` / constraint violations** → `model-configuration.md` + `stack-setup.md` (constraints + merge policy)
- **Batch operations not updating UI** → `persistent-history.md` + `batch-operations.md`
- **CloudKit schema/sync issues** → `cloudkit-integration.md`
- **Memory grows during fetch** → `performance.md` + `fetch-requests.md`

## Verification checklist (when changing Core Data code)

- Confirm the context matches the work (UI vs background).
- Ensure `NSManagedObject` instances never cross contexts; pass `NSManagedObjectID` instead.
- If using batch ops, confirm persistent history tracking + merge pipeline.
- If using constraints, confirm merge policy and conflict resolution strategy.
- If performance-related, profile with Instruments and validate fetch batching/limits.

## Reference files in this subtree

- `_index.md` (navigation, problem→file lookup, file statistics)
- `stack-setup.md`
- `saving.md`
- `fetch-requests.md`
- `threading.md`
- `concurrency.md`
- `batch-operations.md`
- `persistent-history.md`
- `model-configuration.md`
- `migration.md`
- `cloudkit-integration.md`
- `performance.md`
- `testing.md`
- `project-audit.md`
- `glossary.md`
