---
name: swift-data
description: 'Use for ANY Swift/iOS/macOS data persistence work: persisting data, models, or a database; Core Data (NSManagedObject, NSPersistentContainer, NSFetchedResultsController, stack setup, threading, migrations, NSPersistentCloudKitContainer/CloudKit sync); SwiftData (@Model, @Query, @Relationship, ModelContainer, predicates); SQLiteData (@Table, @FetchAll, @FetchOne, @Fetch macros, CloudKit private-database sync); GRDB (raw SQL, DatabaseQueue/DatabasePool, complex joins, window functions, ValueObservation). Also use when deciding WHICH persistence framework to use, migrating between them, or reviewing/writing/debugging persistence code in an existing project. Consolidates four prior skills (core-data-expert, swiftdata-pro, sqlite-data, grdb) into one — always load this first for persistence work rather than guessing at framework-specific APIs from memory.'
---

# Swift Data — persistence router

Four different Swift persistence frameworks solve overlapping problems in incompatible ways. Getting this wrong costs real time: Core Data's `NSManagedObjectID` rules don't apply to SwiftData, SwiftData's `#Predicate` macro doesn't support what GRDB's raw SQL does, and SQLiteData's macros aren't GRDB's query interface. Rather than guessing from general Swift/SQL knowledge, identify the framework first, then read the matching reference below — it has the framework's actual rules, crash patterns, and idioms in depth.

This skill is a **router**, not a rulebook. Each reference file below carries the full content of what used to be a separate skill (`core-data-expert`, `swiftdata-pro`, `sqlite-data`, `grdb`) — nothing was cut, only reorganized.

## Step 1: identify the framework

Look at the project (imports, `Package.swift`/`.xcodeproj` dependencies, existing model files) before assuming. Signals:

| See this | Framework |
|---|---|
| `import CoreData`, `NSManagedObject`, `.xcdatamodeld`, `NSPersistentContainer` | **Core Data** |
| `import SwiftData`, `@Model`, `@Query`, `ModelContainer` | **SwiftData** |
| `import SQLiteData`, `@Table`, `@FetchAll`, `@FetchOne`, `@Fetch` | **SQLiteData** |
| `import GRDB`, `DatabaseQueue`, `DatabasePool`, raw SQL strings, `Record` protocols | **GRDB** |

If there's no existing code (greenfield decision), use this rule of thumb:

- **Default for new SwiftUI apps, iOS 17+**: SwiftData — least boilerplate, native `@Query`/`@Model`.
- **Need CloudKit sync but want type-safe `@Table` macros over raw SQL**: SQLiteData.
- **Need raw SQL, complex joins (4+ tables), window functions, or reactive `ValueObservation`, or are dropping down from SQLiteData for performance**: GRDB.
- **Existing large Core Data codebase, or need NSFetchedResultsController-style diffable UIKit integration**: stay on Core Data — don't migrate speculatively.
- **Have an existing app already committed to one of the four**: stay on it unless the user explicitly asks about migrating. Migrating persistence layers is high-risk; don't suggest it unprompted.

Two frameworks *coexist* deliberately in some codebases: SQLiteData for most tables, GRDB for the handful of queries where SQLiteData's macros aren't expressive enough. See `references/grdb.md` → "When to Use GRDB vs SQLiteData" for the split.

## Step 2: read the matching reference

- **Core Data** → `references/core-data/overview.md` (start here — triage template, routing map, common-error lookup), then the specific file it points to (`stack-setup.md`, `fetch-requests.md`, `threading.md`, `concurrency.md`, `batch-operations.md`, `persistent-history.md`, `model-configuration.md`, `migration.md`, `cloudkit-integration.md`, `performance.md`, `testing.md`, `project-audit.md`, `glossary.md`). `references/core-data/_index.md` has a full problem→file lookup table.
- **SwiftData** → `references/swiftdata.md` — review process, core instructions, output format, then sections on core rules, predicates, CloudKit, indexing (iOS 18+), and class inheritance (iOS 26+).
- **SQLiteData** → `references/sqlitedata.md` — section guide up top routes to table models, query basics/advanced, writes, SwiftUI/UIKit view integration, migrations, CloudKit sync, dependency injection, testing, and advanced optimization.
- **GRDB** → `references/grdb.md` — getting started, queries, ValueObservation, migrations, performance, plus the GRDB-vs-SQLiteData comparison table.

Don't load more than one framework's reference unless the task genuinely spans two (e.g., a SQLiteData app with one GRDB escape-hatch query, or a Core Data → SwiftData migration question).

## Cross-cutting behavior contract

These apply regardless of which framework is in play — they're *why*, not just *what*, because the underlying reason (object lifetime, actor isolation, schema evolution) recurs across all four:

1. **Never let a model/record instance cross a concurrency boundary it doesn't own.** Core Data: pass `NSManagedObjectID`, not `NSManagedObject`, across contexts. SwiftData: `ModelContext` and model instances must never cross actor boundaries — pass `PersistentIdentifier` and re-fetch. GRDB/SQLiteData: reads and writes go through the database connection/queue, not cached record instances shared across threads. The common failure mode is the same crash shape (concurrent access, stale reference) with a framework-specific fix.
2. **Determine the deployment target before proposing an API.** All four gate features by OS version (Core Data staged migration iOS 17+, SwiftData indexing iOS 18+/class inheritance iOS 26+, etc.) — check before recommending, don't assume latest.
3. **Migrations are not optional busywork.** A schema change without a matching migration is a runtime crash waiting to happen in every one of these frameworks, not just a style nit — treat "did you add a migration" as a required check on any model/table change.
4. **CloudKit constraints bind the schema, not just the code.** Core Data's production CloudKit schema is immutable once shipped; SQLiteData's CloudKit sync needs full `SyncDelegate` implementation for production. Flag CloudKit involvement early — it changes what's safe to change later.
5. **Profile before optimizing.** "Probably slow" isn't a diagnosis in any of these frameworks — use Instruments (Core Data), or `EXPLAIN QUERY PLAN` (GRDB/SQLiteData raw queries) before restructuring for performance.

## Verification checklist (any framework, before calling persistence work done)

- Confirmed which framework this project actually uses (didn't assume from the request wording).
- Model/record instances aren't crossing threads or actors by reference.
- Schema changes have a corresponding migration.
- If CloudKit is involved, sync/schema implications were considered, not just the local-store change.
- Performance claims are profiled, not guessed.
- Read the framework-specific reference file rather than answering from general SQL/ORM knowledge.

## Reference tree

```
references/
  core-data/            (Core Data — full former core-data-expert skill, 15 files)
    overview.md          entry point: triage template, routing map, common errors, verification
    _index.md             navigation index, problem→file lookup
    stack-setup.md, saving.md, fetch-requests.md, threading.md, concurrency.md,
    batch-operations.md, persistent-history.md, model-configuration.md, migration.md,
    cloudkit-integration.md, performance.md, testing.md, project-audit.md, glossary.md
  swiftdata.md           (SwiftData — full former swiftdata-pro skill)
  sqlitedata.md          (SQLiteData — full former sqlite-data skill)
  grdb.md                (GRDB — full former grdb skill)
assets/
  swiftdata-pro-icon.png, swiftdata-pro-icon.svg
agents/
  openai.yaml
```
