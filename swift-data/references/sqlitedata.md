# SQLiteData

Full content of the former standalone `sqlite-data` skill. SQLiteData provides type-safe SQLite access through Swift macros (`@Table`, `@FetchAll`, `@FetchOne`), simplifying database modeling and queries while handling CloudKit sync, migrations, and async patterns automatically.

## Section guide

**Prefer reading a section over skipping it if there's even a small chance the content may be required** — it's better to have the context than to miss a pattern or make a mistake.

| Section | Read When |
|-----------|-----------|
| [Table Models](#table-models) | Defining tables with `@Table`, setting up primary keys, columns, or enums |
| [Query Basics](#query-basics) | Using `@FetchAll`, `@FetchOne`, `@Selection`, filtering, ordering, or joins |
| [Query Advanced Patterns](#query-advanced-patterns) | Using `@Fetch` with `FetchKeyRequest`, dynamic queries, recursive CTEs, or direct reads |
| [Database Writes](#database-writes) | Inserting, updating, upserting, deleting records, or managing transactions |
| [SwiftUI View Integration](#swiftui-view-integration) | Using `@FetchAll`/`@FetchOne` in SwiftUI views, `@Observable` models, or animations |
| [View Integration Patterns](#view-integration-patterns) | UIKit integration, dynamic query loading, TCA integration, or `observe {}` |
| [Database Migrations](#database-migrations) | Creating database migrations with `DatabaseMigrator` or `#sql()` macro |
| [CloudKit Sync](#cloudkit-sync) | Setting up CloudKit private database sync, sharing, or sync delegates |
| [Dependency Injection](#dependency-injection) | Injecting database/sync engine via `@Dependency`, bootstrap patterns, or TCA integration |
| [Testing](#testing) | Setting up test databases, seeding data, or writing assertions for SQLite code |
| [Advanced Query Features](#advanced-query-features) | Implementing triggers, custom database functions, or full-text search (FTS5) |
| [Advanced Optimization & Aggregation](#advanced-optimization--aggregation) | Performance tuning, indexes, custom aggregates, JSON aggregation, or self-joins |
| [Schema Composition](#schema-composition) | Using `@Selection` column groups, single-table inheritance, or database views |

## Core Workflow

When working with SQLiteData:
1. Define table models with `@Table` macro
2. Use `@FetchAll`/`@FetchOne` property wrappers in views or `@Observable` models
3. Access database via `@Dependency(\.defaultDatabase)`
4. Perform writes in `database.write { }` transactions
5. Set up migrations before first use

## Common Mistakes

1. **N+1 query patterns** — Loading records one-by-one in a loop (e.g., fetching user then fetching all their posts separately) kills performance. Use joins or batch fetches instead.

2. **Missing migrations on schema changes** — Modifying `@Table` without creating a migration causes crashes at runtime. Always create migrations for schema changes before deploying.

3. **Improper transaction handling** — Long-running transactions outside of `database.write { }` block can cause deadlocks or data loss. Keep write blocks short and focused.

4. **Ignoring CloudKit sync delegates** — Setting up CloudKit sync without implementing `SyncDelegate` means you miss error handling and conflict resolution. Implement all delegate methods for production.

5. **Over-fetching in SwiftUI views** — Using `@FetchAll` without filtering/limiting can load thousands of records, freezing the UI. Use predicates, limits, and sorting to keep in-memory footprint small.

---


## Table Models

Patterns for defining database tables using the `@Table` macro.

### Basic Table Definition

```swift
@Table
nonisolated struct SyncUp: Hashable, Identifiable {
  let id: UUID
  var title = ""
  var seconds: Int = 60 * 5
  var theme: Theme = .bubblegum
}
```

**Requirements:**
- Marked with `@Table` macro
- `nonisolated` for Sendable conformance
- `Identifiable` conformance (primary key defaults to `id` property)
- `Hashable` conformance

### Primary Keys

#### Auto-Generated Primary Key

By default, a property named `id` is used as the primary key:

```swift
@Table
nonisolated struct Meeting: Hashable, Identifiable {
  let id: UUID  // Automatically becomes primary key
  var date: Date
  var notes: String
}
```

#### Custom Primary Key

Use `@Column(primaryKey: true)` for custom primary keys:

```swift
@Table
nonisolated struct Tag: Hashable, Identifiable {
  @Column(primaryKey: true)
  var title: String
  var id: String { title }
}
```

#### Composite Primary Key

For junction tables or tables with composite keys:

```swift
@Table
nonisolated struct RemindersListAsset: Hashable, Identifiable {
  @Column(primaryKey: true)
  let remindersListID: RemindersList.ID
  var coverImage: Data?
  var id: RemindersList.ID { remindersListID }
}
```

### Custom Column Types

Use `@Column(as:)` for custom type representations:

```swift
@Table
nonisolated struct RemindersList: Hashable, Identifiable {
  let id: UUID
  @Column(as: Color.HexRepresentation.self)
  var color: Color = Self.defaultColor
  var title = ""
}

extension Color {
  struct HexRepresentation: ColumnRepresentable {
    // Implementation for converting Color to/from hex string
  }
}
```

### Foreign Keys

Define foreign key relationships by referencing another table's ID type:

```swift
@Table
nonisolated struct Attendee: Hashable, Identifiable {
  let id: UUID
  var name = ""
  var syncUpID: SyncUp.ID  // Foreign key to SyncUp table
}
```

### Draft Types

The `@Table` macro auto-generates a `.Draft` type for insertions:

```swift
// Auto-generated:
extension SyncUp {
  struct Draft {
    var id: UUID = UUID()
    var title = ""
    var seconds: Int = 60 * 5
    var theme: Theme = .bubblegum
  }
}

// Usage:
SyncUp.insert {
  SyncUp.Draft(
    title: "Daily Standup",
    seconds: 900
  )
}.execute(db)
```

Make Draft types conform to Identifiable when needed:

```swift
extension SyncUp.Draft: Identifiable {}
```

### Nested Enums

Enums conforming to `QueryBindable` can be used as column types:

```swift
@Table
nonisolated struct Reminder: Hashable, Identifiable {
  let id: UUID
  var priority: Priority?
  var status: Status = .incomplete

  enum Priority: Int, QueryBindable {
    case low = 1
    case medium
    case high
  }

  enum Status: Int, QueryBindable {
    case completed = 1
    case completing = 2
    case incomplete = 0
  }
}
```

### Computed Properties

Add computed properties for convenience (not stored in database):

```swift
@Table
nonisolated struct Reminder: Hashable, Identifiable {
  let id: UUID
  var status: Status

  var isCompleted: Bool {
    status != .incomplete
  }

  enum Status: Int, QueryBindable {
    case incomplete = 0
    case completed = 1
    case completing = 2
  }
}
```

### TableColumns Extensions

Extend `TableColumns` for computed query expressions:

```swift
nonisolated extension Reminder.TableColumns {
  var isCompleted: some QueryExpression<Bool> {
    status.neq(Reminder.Status.incomplete)
  }

  var isPastDue: some QueryExpression<Bool> {
    @Dependency(\.date.now) var now
    return !isCompleted && #sql("coalesce(date(\(dueDate)) < date(\(now)), 0)")
  }

  var isToday: some QueryExpression<Bool> {
    @Dependency(\.date.now) var now
    return !isCompleted && #sql("coalesce(date(\(dueDate)) = date(\(now)), 0)")
  }
}

// Usage in queries:
Reminder.where { $0.isPastDue }.fetchAll(db)
```

### Static Query Helpers

Define static properties for common queries:

```swift
extension Reminder {
  static let incomplete = Self.where { !$0.isCompleted }

  static let withTags = group(by: \.id)
    .leftJoin(ReminderTag.all) { $0.id.eq($1.reminderID) }
    .leftJoin(Tag.all) { $1.tagID.eq($2.primaryKey) }
}

// Usage:
let incompleteTasks = try Reminder.incomplete.fetchAll(db)
```

---

## Query Basics

Patterns for fetching data using `@FetchAll`, `@FetchOne`, and `@Selection`.

### When to Use Which

| Wrapper | Use For | Example |
|---------|---------|---------|
| `@FetchAll` | Inline queries returning **multiple records** | `@FetchAll(Item.order { $0.createdAt.desc() }) var items` |
| `@FetchOne` | Inline queries returning **single record or aggregate** | `@FetchOne(Item.where { $0.isActive }) var activeItem` |
| `@Fetch` | **FetchKeyRequest only** - when you need data transformation | `@Fetch(ComplexRequest()) var result` |

**Common mistake**: Using `@Fetch` for simple queries. It requires `FetchKeyRequest` conformance.

Only create a `FetchKeyRequest` struct when you need to:
- Transform data with `.map()` after fetching
- Perform multiple database operations in one transaction
- Return a custom result type that differs from the raw query

For simple ordering, filtering, or joins without transformation → use `@FetchAll` or `@FetchOne`.

### @FetchAll - Multiple Records

Fetch multiple records with automatic SwiftUI updates:

```swift
@Observable
class CountersListModel {
  @ObservationIgnored
  @FetchAll var counters: [Counter]
}
```

#### With Query Ordering

```swift
@FetchAll(
  Counter.order(by: \.id),
  animation: .default
)
var counters
```

#### With Filtering

```swift
@FetchAll(
  Reminder.where { !$0.isCompleted }
    .order(by: \.position)
)
var incompleteTasks
```

#### With Joins

```swift
@FetchAll(
  RemindersList
    .group(by: \.id)
    .order(by: \.position)
    .leftJoin(Reminder.all) { $0.id.eq($1.remindersListID) && !$1.isCompleted }
    .leftJoin(SyncMetadata.all) { $0.syncMetadataID.eq($2.id) }
    .select {
      ReminderListState.Columns(
        remindersCount: $1.id.count(),
        remindersList: $0,
        share: $2.share
      )
    },
  animation: .default
)
var remindersLists
```

### @FetchOne - Single Value or Aggregate

Fetch a single record or aggregate value:

```swift
@FetchOne(Fact.count(), animation: .default)
var factsCount = 0
```

#### Multiple Aggregates with @Selection

```swift
@FetchOne(
  Reminder.select {
    Stats.Columns(
      allCount: $0.count(filter: !$0.isCompleted),
      flaggedCount: $0.count(filter: $0.isFlagged && !$0.isCompleted),
      scheduledCount: $0.count(filter: $0.isScheduled),
      todayCount: $0.count(filter: $0.isToday)
    )
  }
)
var stats = Stats()

@Selection
struct Stats {
  var allCount = 0
  var flaggedCount = 0
  var scheduledCount = 0
  var todayCount = 0
}
```

### @Selection - Custom Result Types

Define custom result types for complex queries:

```swift
@Selection
struct ReminderListState: Identifiable, Hashable {
  var remindersList: RemindersList
  var remindersCount: Int
  @Column(as: CKShare?.self)
  var share: CKShare?

  var id: RemindersList.ID { remindersList.id }
}
```

Use with queries:

```swift
@FetchAll(
  RemindersList
    .leftJoin(Reminder.all) { $0.id.eq($1.remindersListID) }
    .select { list, reminder in
      ReminderListState.Columns(
        remindersList: list,
        remindersCount: reminder.id.count()
      )
    }
)
var remindersLists
```

### Query Building Blocks

#### Filtering

```swift
// Simple equality
Reminder.where { $0.status.eq(.incomplete) }

// Negation
Reminder.where { !$0.isCompleted }

// Comparisons
Reminder.where { $0.priority.gt(.low) }

// In array
Tag.where { $0.title.in(["Work", "Personal"]) }

// Pattern matching
Fact.where { $0.body.contains(searchText) }

// Null checks
Reminder.where { $0.dueDate.isNot(nil) }
```

#### Ordering

```swift
// Single column
Counter.order(by: \.id)

// Multiple columns with direction
Reminder
  .order { $0.dueDate.desc() }
  .order { $0.position }
```

#### Grouping

```swift
RemindersList
  .group(by: \.id)
  .leftJoin(Reminder.all) { $0.id.eq($1.remindersListID) }
  .select { list, reminder in
    ListSummary.Columns(
      list: list,
      count: reminder.id.count()
    )
  }
```

#### Joining

```swift
// Left join
Tag
  .leftJoin(ReminderTag.all) { $0.primaryKey.eq($1.tagID) }
  .leftJoin(Reminder.all) { $1.reminderID.eq($2.id) }

// With conditions
RemindersList
  .leftJoin(Reminder.all) {
    $0.id.eq($1.remindersListID) && !$1.isCompleted
  }
```

#### Having

Filter grouped results:

```swift
Tag
  .withReminders
  .having { $2.count().gt(0) }  // Only tags with reminders
  .select { tag, _, _ in tag }
```

#### Count

```swift
// Total count
let total = try Reminder.fetchCount(db)

// Filtered count
let incomplete = try Reminder.where { !$0.isCompleted }.fetchCount(db)

// Conditional count in select
Reminder.select {
  Stats.Columns(
    allCount: $0.count(filter: !$0.isCompleted),
    flaggedCount: $0.count(filter: $0.isFlagged)
  )
}
```

---

## Query Advanced Patterns

FetchKeyRequest, dynamic queries, direct database access, and recursive CTEs.

### @Fetch with FetchKeyRequest

For complex queries that need multiple database operations in a single transaction:

```swift
@Fetch(Facts(), animation: .default)
private var facts = Facts.Value()

private struct Facts: FetchKeyRequest {
  var query = ""

  struct Value {
    var facts: [Fact] = []
    var searchCount = 0
    var totalCount = 0
  }

  func fetch(_ db: Database) throws -> Value {
    let search = Fact
      .where { $0.body.contains(query) }
      .order { $0.id.desc() }

    return try Value(
      facts: search.fetchAll(db),
      searchCount: search.fetchCount(db),
      totalCount: Fact.fetchCount(db)
    )
  }
}
```

### Dynamic Query Loading

Update queries dynamically using the projected value:

```swift
@Fetch(SearchRequest(text: ""), animation: .default)
var searchResults = SearchResults()

// In view:
.task(id: searchText) {
  try await $searchResults.load(
    SearchRequest(text: searchText),
    animation: .default
  )
}
```

### Reading from Database Directly

For non-reactive queries in imperative code:

```swift
@Dependency(\.defaultDatabase) var database

// Read transaction
let counters = try database.read { db in
  try Counter.order(by: \.id).fetchAll(db)
}

// Fetch single record
let counter = try database.read { db in
  try Counter.find(id).fetchOne(db)
}
```

### Static Fetch Helpers (v1.4+)

Convenient static methods for common fetches:

```swift
// Fetch all records
let items = try Item.fetchAll(db)

// Fetch with query
let active = try Item.where { !$0.isArchived }.fetchAll(db)

// Find by primary key
let item = try Item.find(db, key: id)

// Fetch count
let total = try Item.fetchCount(db)
```

### Recursive CTEs

Query hierarchical data like trees or org charts:

```swift
@Table
nonisolated struct Category: Identifiable {
    let id: UUID
    var name = ""
    var parentID: UUID?  // Self-referential
}

// Get all descendants of a category
let descendants = try With {
    // Base case: start with root
    Category.where { $0.id.eq(rootCategoryId) }
} recursiveUnion: { cte in
    // Recursive case: join children to CTE
    Category.all
        .join(cte) { $0.parentID.eq($1.id) }
        .select { $0 }
} query: { cte in
    cte.order(by: \.name)
}
.fetchAll(db)
```

#### Walking Up the Tree (Ancestors)

```swift
let ancestors = try With {
    Category.where { $0.id.eq(childCategoryId) }
} recursiveUnion: { cte in
    Category.all
        .join(cte) { $0.id.eq($1.parentID) }
        .select { $0 }
} query: { cte in
    cte.all
}
.fetchAll(db)
```

#### Threaded Comments with Depth

```swift
let thread = try With {
    Comment
        .where { $0.parentID.is(nil) && $0.postID.eq(postId) }
        .select { ($0, 0) }  // depth = 0 for root
} recursiveUnion: { cte in
    Comment.all
        .join(cte) { $0.parentID.eq($1.id) }
        .select { ($0, $1.depth + 1) }
} query: { cte in
    cte.order { ($0.depth, $0.createdAt) }
}
.fetchAll(db)
```

### Best Practices

1. **Use `@FetchAll`/`@FetchOne`** for simple queries - avoid `FetchKeyRequest` overhead
2. **Use `FetchKeyRequest`** only when you need multiple fetches or data transformation
3. **Use dynamic loading** with `$property.load()` for search/filter scenarios
4. **Prefer reactive queries** (`@FetchAll`) over imperative reads when possible
5. **Use recursive CTEs** for hierarchical data instead of multiple queries

---

## Database Writes

Patterns for inserting, updating, upserting, and deleting records.

### Insert Operations

#### Single Insert

```swift
try database.write { db in
  try Counter.insert {
    Counter.Draft()
  }.execute(db)
}
```

#### Insert with Values

```swift
try database.write { db in
  try Fact.insert {
    Fact.Draft(body: "An interesting fact")
  }.execute(db)
}
```

#### Batch Insert

```swift
try database.write { db in
  try Attendee.insert {
    for attendee in attendees {
      Attendee.Draft(
        id: attendee.id,
        name: attendee.name,
        syncUpID: syncUpID
      )
    }
  }.execute(db)
}
```

#### Insert with Return Value

```swift
try database.write { db in
  let reminderID = try Reminder.insert {
    Reminder.Draft(
      title: "Buy groceries",
      remindersListID: listID
    )
  }
  .returning(\.id)
  .fetchOne(db)!
}
```

### Update Operations

#### Simple Update

```swift
try database.write { db in
  try Counter.find(counter.id).update {
    $0.count += 1
  }.execute(db)
}
```

#### Update with Multiple Fields

```swift
try database.write { db in
  try Reminder.find(reminderID).update {
    $0.title = "Updated title"
    $0.dueDate = Date()
    $0.priority = .high
  }.execute(db)
}
```

#### Batch Update with Filter

```swift
try database.write { db in
  try Reminder
    .where { $0.remindersListID.eq(listID) }
    .update {
      $0.position += 1
    }
    .execute(db)
}
```

#### Update with Case Expression

For conditional updates, use `Case().when().else()`:

```swift
extension Updates<Reminder> {
  mutating func toggleStatus() {
    self.status = Case(self.status)
      .when(#bind(.incomplete), then: #bind(.completing))
      .else(#bind(.incomplete))
  }
}

// Usage:
try database.write { db in
  try Reminder.find(id).update {
    $0.toggleStatus()
  }.execute(db)
}
```

#### Batch Position Update

```swift
try database.write { db in
  let ids = [
    (element: UUID(), offset: 0),
    (element: UUID(), offset: 1),
    (element: UUID(), offset: 2)
  ]
  let (first, rest) = (ids.first!, ids.dropFirst())

  try RemindersList.update {
    $0.position = rest.reduce(
      Case($0.id).when(first.element, then: first.offset)
    ) { cases, id in
      cases.when(id.element, then: id.offset)
    }
    .else($0.position)
  }.execute(db)
}
```

### Upsert Operations

Insert a record or update if it already exists:

```swift
try database.write { db in
  let syncUpID = try SyncUp.upsert { syncUp }
    .returning(\.id)
    .fetchOne(db)!
}
```

#### Upsert with Optional Return

```swift
try database.write { db in
  let remindersListID = try RemindersList
    .upsert { remindersList }
    .returning(\.id)
    .fetchOne(db)

  guard let remindersListID else { return }

  // Continue with dependent operations
}
```

#### Upsert Pattern for Updates

Common pattern: upsert main record, then replace child records:

```swift
try database.write { db in
  // Upsert parent
  let syncUpID = try SyncUp.upsert { syncUp }.returning(\.id).fetchOne(db)!

  // Delete existing children
  try Attendee.where { $0.syncUpID == syncUpID }.delete().execute(db)

  // Insert new children
  try Attendee.insert {
    for attendee in attendees {
      Attendee.Draft(
        id: attendee.id,
        name: attendee.name,
        syncUpID: syncUpID
      )
    }
  }.execute(db)
}
```

### Delete Operations

#### Delete by ID

```swift
try database.write { db in
  try Counter.find(counterID).delete().execute(db)
}
```

#### Delete Multiple Records

```swift
try database.write { db in
  for index in indexSet {
    try Counter.find(counters[index].id).delete().execute(db)
  }
}
```

#### Delete with Filter

```swift
try database.write { db in
  try Tag
    .where { $0.title.in(tagTitles) }
    .delete()
    .execute(db)
}
```

#### Delete by ID Array

```swift
try database.write { db in
  let ids = indices.map { facts[$0].id }
  try Fact
    .where { $0.id.in(ids) }
    .delete()
    .execute(db)
}
```

#### Conditional Delete

```swift
try database.write { db in
  try Reminder
    .where { $0.status.eq(.completed) && $0.completedDate.lt(cutoffDate) }
    .delete()
    .execute(db)
}
```

### Error Handling

Wrap database writes with error reporting:

```swift
withErrorReporting {
  try database.write { db in
    // Write operations
  }
}
```

For async context:

```swift
await withErrorReporting {
  try await database.write { db in
    // Write operations
  }
}
```

### Transaction Guarantees

All operations within `database.write { }` execute in a single transaction:

```swift
try database.write { db in
  // These all succeed or all fail together
  try Counter.insert { Counter.Draft() }.execute(db)
  try Counter.find(otherID).delete().execute(db)
  try Counter.find(thirdID).update { $0.count = 0 }.execute(db)
}
```

---

## SwiftUI View Integration

Patterns for integrating database queries with SwiftUI views and @Observable models.

### SwiftUI Views

#### Direct in View

Use `@FetchAll` and `@FetchOne` directly in SwiftUI views:

```swift
struct CountersListView: View {
  @FetchAll var counters: [Counter]
  @FetchOne(Counter.count()) var countersCount = 0

  var body: some View {
    List {
      Text("Total: \(countersCount)")
      ForEach(counters) { counter in
        Text("\(counter.count)")
      }
    }
  }
}
```

#### With Query and Animation

```swift
struct SwiftUIDemo: View {
  @FetchAll(Fact.order { $0.id.desc() }, animation: .default)
  private var facts

  @FetchOne(Fact.count(), animation: .default)
  var factsCount = 0

  var body: some View {
    List {
      Section {
        Text("Facts: \(factsCount)")
          .font(.largeTitle)
          .contentTransition(.numericText(value: Double(factsCount)))
      }
      Section {
        ForEach(facts) { fact in
          Text(fact.body)
        }
      }
    }
  }
}
```

#### With Complex Queries

```swift
@FetchAll(
  RemindersList
    .group(by: \.id)
    .order(by: \.position)
    .leftJoin(Reminder.all) { $0.id.eq($1.remindersListID) && !$1.isCompleted }
    .select {
      ListSummary.Columns(
        list: $0,
        incompleteCount: $1.id.count()
      )
    },
  animation: .default
)
var remindersLists
```

### @Observable Models

Use `@ObservationIgnored` to prevent observation of the fetch wrapper itself:

```swift
@Observable
@MainActor
class Model {
  @ObservationIgnored
  @FetchAll(Fact.order { $0.id.desc() }, animation: .default)
  var facts

  @ObservationIgnored
  @FetchOne(Fact.count(), animation: .default)
  var factsCount = 0

  @ObservationIgnored
  @Dependency(\.defaultDatabase) private var database

  func deleteFact(indices: IndexSet) {
    withErrorReporting {
      try database.write { db in
        let ids = indices.map { facts[$0].id }
        try Fact.where { $0.id.in(ids) }.delete().execute(db)
      }
    }
  }
}
```

#### In SwiftUI View

```swift
struct ObservableModelDemo: View {
  @State private var model = Model()

  var body: some View {
    List {
      Text("Facts: \(model.factsCount)")
      ForEach(model.facts) { fact in
        Text(fact.body)
      }
      .onDelete { indices in
        model.deleteFact(indices: indices)
      }
    }
  }
}
```

### Animations

#### Default Animation

```swift
@FetchAll(Counter.all, animation: .default)
var counters
```

#### Custom Animation

```swift
@FetchAll(
  Reminder.where { !$0.isCompleted },
  animation: .spring(response: 0.3, dampingFraction: 0.7)
)
var incompleteTasks
```

#### Numeric Transitions

Use `.contentTransition()` for smooth number updates:

```swift
Text("Count: \(factsCount)")
  .contentTransition(.numericText(value: Double(factsCount)))
```

### Best Practices

1. **Use `@ObservationIgnored`** on `@FetchAll`/`@FetchOne` in `@Observable` classes
2. **Always specify `animation:`** parameter for smooth UI updates
3. **Use `.contentTransition()`** for numeric value animations
4. **Wrap deletes in `withErrorReporting`** for consistent error handling
5. **Mark `@Observable` models as `@MainActor`** when used with SwiftUI

---

## View Integration Patterns

UIKit integration, dynamic queries, and TCA patterns.

### UIKit Integration

Use `observe {}` block to react to database changes:

```swift
final class UIKitCaseStudyViewController: UICollectionViewController {
  private var dataSource: UICollectionViewDiffableDataSource<Section, Fact>!

  @FetchAll(Fact.order { $0.id.desc() }, animation: .default)
  private var facts

  @Dependency(\.defaultDatabase) var database

  override func viewDidLoad() {
    super.viewDidLoad()

    // Setup data source
    dataSource = UICollectionViewDiffableDataSource<Section, Fact>(
      collectionView: collectionView
    ) { collectionView, indexPath, item in
      // Cell configuration
    }

    // Observe database changes
    observe { [weak self] in
      guard let self else { return }
      var snapshot = NSDiffableDataSourceSnapshot<Section, Fact>()
      snapshot.appendSections([.facts])
      snapshot.appendItems(facts, toSection: .facts)
      dataSource.apply(snapshot)
    }
  }
}
```

### Dynamic Query Loading

Update queries dynamically using the projected value:

```swift
struct DynamicQueryDemo: View {
  @Fetch(Facts(), animation: .default)
  private var facts = Facts.Value()

  @State var query = ""

  var body: some View {
    List {
      ForEach(facts.facts) { fact in
        Text(fact.body)
      }
    }
    .searchable(text: $query)
    .task(id: query) {
      await withErrorReporting {
        try await $facts.load(Facts(query: query), animation: .default)
      }
    }
  }

  private struct Facts: FetchKeyRequest {
    var query = ""
    struct Value {
      var facts: [Fact] = []
    }
    func fetch(_ db: Database) throws -> Value {
      try Value(
        facts: Fact.where { $0.body.contains(query) }.fetchAll(db)
      )
    }
  }
}
```

### Manual Refresh

Manually trigger a query refresh in @Observable models:

```swift
@Observable
class SearchModel {
  @ObservationIgnored
  @Fetch(SearchRequest(text: ""), animation: .default)
  var results = SearchResults()

  var searchText = "" {
    didSet {
      Task {
        try await $results.load(
          SearchRequest(text: searchText),
          animation: .default
        )
      }
    }
  }
}
```

### TCA Integration

Use `@Fetch`/`@FetchOne` directly in TCA `@ObservableState` for reactive queries:

```swift
@ObservableState
struct State: Equatable {
    @Fetch(ItemsRequest()) var items: [Item] = []
    @FetchOne(Bundle.where { $0.isActive }) var activeBundle: Bundle?
}
```

#### FetchKeyRequest for Complex Queries

```swift
struct ItemsRequest: FetchKeyRequest {
    typealias Value = [Item]

    func fetch(_ db: Database) throws -> [Item] {
        try Item
            .where { $0.isArchived == false }
            .order { $0.createdAt.desc() }
            .join(ItemDetail.all) { $1.id.eq($0.id) }
            .select {
                Item.Columns(
                    id: $0.id,
                    title: $1.title,
                    createdAt: $0.createdAt
                )
            }
            .fetchAll(db)
    }
}
```

#### Anti-Pattern: Imperative Fetch Functions

```swift
// WRONG - Creates unnecessary Effect/Action boilerplate
// Requires manual refetch after every mutation
private func fetchItems() -> Effect<Action> {
    .run { send in
        let items = try await database.read { db in ... }
        await send(.itemsLoaded(items))
    }
}

case .view(.onAppear):
    return fetchItems()  // Must call on appear

case .view(.onItemDeleted(let id)):
    return .run { send in
        try await database.deleteItem(id)
        // Must manually refetch after mutation!
        let items = try await database.read { ... }
        await send(.itemsLoaded(items))
    }
```

```swift
// RIGHT - Use @Fetch, mutations auto-refresh
@ObservableState
struct State: Equatable {
    @Fetch(ItemsRequest()) var items: [Item] = []
}

case .view(.onAppear):
    return .none  // Nothing needed - @Fetch observes automatically

case .view(.onItemDeleted(let id)):
    return .run { _ in
        try await database.deleteItem(id)
        // No refetch needed - @Fetch updates automatically
    }
```

### Best Practices

1. **Use `observe {}`** in UIKit for reactive updates
2. **Use dynamic loading** with `$property.load()` for search/filter scenarios
3. **Use `@Fetch`/`@FetchOne` in TCA State** - avoid imperative fetch functions
4. **Avoid imperative patterns** - let the property wrappers handle refetching
5. **Use `.task(id:)`** for search queries that change based on user input

---

## Database Migrations

Patterns for managing database schema with `DatabaseMigrator`.

### Migration Strategy

**IMPORTANT:** Before creating database migrations, clarify the app's development stage with the user.

#### During Active Development (Pre-Release)

If the app has not been released to users yet:
- **Do NOT create new migration files** for schema changes
- Instead, **update existing migrations in place**
- Ask the user: "This app appears to be in development. Should I update the existing migration, or create a new one?"

#### After Release (Production)

Once an app is released:
- **Always create new migration files** for schema changes
- Never modify existing migrations (users have data in the old schema)
- Migrations must be additive and backwards-compatible

#### Clarifying Question

When schema changes are needed, ask:

> "Is this app already released to users, or still in development?
> - **In development** — I'll update the existing schema directly
> - **Released** — I'll create a new migration to preserve user data"

### Basic Migration Setup

```swift
var migrator = DatabaseMigrator()

#if DEBUG
  migrator.eraseDatabaseOnSchemaChange = true
#endif

migrator.registerMigration("Create initial tables") { db in
  try #sql(
    """
    CREATE TABLE "counters" (
      "id" TEXT PRIMARY KEY NOT NULL,
      "count" INTEGER NOT NULL DEFAULT 0
    ) STRICT
    """
  ).execute(db)
}

try migrator.migrate(database)
```

### Using #sql() Macro

The `#sql()` macro provides type-safe SQL with interpolation:

```swift
migrator.registerMigration("Create users table") { db in
  try #sql(
    """
    CREATE TABLE "users" (
      "id" TEXT PRIMARY KEY NOT NULL ON CONFLICT REPLACE DEFAULT (uuid()),
      "name" TEXT NOT NULL,
      "createdAt" TEXT NOT NULL
    ) STRICT
    """
  ).execute(db)
}
```

#### With Dynamic Values

```swift
migrator.registerMigration("Create remindersLists table") { db in
  let defaultListColor = Color.HexRepresentation(
    queryOutput: RemindersList.defaultColor
  ).hexValue

  try #sql(
    """
    CREATE TABLE "remindersLists" (
      "id" TEXT PRIMARY KEY NOT NULL ON CONFLICT REPLACE DEFAULT (uuid()),
      "color" INTEGER NOT NULL ON CONFLICT REPLACE DEFAULT \(raw: defaultListColor ?? 0),
      "title" TEXT NOT NULL ON CONFLICT REPLACE DEFAULT ''
    ) STRICT
    """
  ).execute(db)
}
```

### STRICT Tables

Use `STRICT` mode for type safety:

```swift
CREATE TABLE "items" (
  "id" TEXT PRIMARY KEY NOT NULL,
  "count" INTEGER NOT NULL,
  "name" TEXT NOT NULL
) STRICT
```

### Foreign Key Constraints

Define foreign keys with cascading deletes:

```swift
migrator.registerMigration("Create attendees table") { db in
  try #sql(
    """
    CREATE TABLE "attendees" (
      "id" TEXT PRIMARY KEY NOT NULL,
      "name" TEXT NOT NULL,
      "syncUpID" TEXT NOT NULL REFERENCES "syncUps"("id") ON DELETE CASCADE
    ) STRICT
    """
  ).execute(db)
}
```

### Multiple Migrations

Register multiple migrations in sequence:

```swift
migrator.registerMigration("Create initial tables") { db in
  // Create tables
}

migrator.registerMigration("Create foreign key indexes") { db in
  try #sql(
    """
    CREATE INDEX IF NOT EXISTS "idx_reminders_remindersListID"
    ON "reminders"("remindersListID")
    """
  ).execute(db)

  try #sql(
    """
    CREATE INDEX IF NOT EXISTS "idx_remindersTags_reminderID"
    ON "remindersTags"("reminderID")
    """
  ).execute(db)
}

try migrator.migrate(database)
```

### FTS5 Virtual Tables

Create full-text search tables:

```swift
migrator.registerMigration("Create FTS5 table") { db in
  try #sql(
    """
    CREATE VIRTUAL TABLE "reminderTexts" USING fts5(
      "title",
      "notes",
      "tags",
      tokenize = 'trigram'
    )
    """
  ).execute(db)
}
```

### Common Column Patterns

#### UUID Primary Key

```swift
"id" TEXT PRIMARY KEY NOT NULL ON CONFLICT REPLACE DEFAULT (uuid())
```

#### Auto-increment Integer

```swift
"id" INTEGER PRIMARY KEY AUTOINCREMENT
```

#### Timestamps

```swift
"createdAt" TEXT NOT NULL
"updatedAt" TEXT
```

#### Booleans

```swift
"isFlagged" INTEGER NOT NULL ON CONFLICT REPLACE DEFAULT 0
```

#### Enums

```swift
"status" INTEGER NOT NULL DEFAULT 0
"priority" INTEGER
```

#### Foreign Keys with Cascade

```swift
"remindersListID" TEXT NOT NULL REFERENCES "remindersLists"("id") ON DELETE CASCADE
```

#### Nullable Fields

```swift
"dueDate" TEXT
"notes" TEXT
"coverImage" BLOB
```

#### Case-Insensitive Text

```swift
"title" TEXT COLLATE NOCASE PRIMARY KEY NOT NULL
```

### Database Configuration

Enable foreign keys and prepare the database:

```swift
var configuration = Configuration()
configuration.foreignKeysEnabled = true
configuration.prepareDatabase { db in
  try db.attachMetadatabase()  // For CloudKit sync
  db.add(function: $myCustomFunction)
}

let database = try SQLiteData.defaultDatabase(configuration: configuration)
```

### Debug Tracing

Enable query tracing in DEBUG builds:

```swift
configuration.prepareDatabase { db in
  #if DEBUG
    db.trace(options: .profile) {
      logger.debug("\($0.expandedDescription)")
    }
  #endif
}
```

### Erase on Schema Change

During development, automatically recreate the database when schema changes:

```swift
#if DEBUG
  migrator.eraseDatabaseOnSchemaChange = true
#endif
```

**Warning:** This deletes all data. Only use during active development.

---

## CloudKit Sync

Patterns for syncing database records with CloudKit private database.

### SyncEngine Setup

#### Basic Setup

```swift
let syncEngine = try SyncEngine(
  for: database,
  tables: Counter.self
)
```

#### Multiple Tables

```swift
let syncEngine = try SyncEngine(
  for: database,
  tables: SyncUp.self, Attendee.self, Meeting.self
)
```

#### With Delegate

```swift
let syncEngine = try SyncEngine(
  for: database,
  tables: RemindersList.self,
  RemindersListAsset.self,
  Reminder.self,
  Tag.self,
  ReminderTag.self,
  delegate: syncEngineDelegate
)
```

### Bootstrap Database with Sync

```swift
extension DependencyValues {
  mutating func bootstrapDatabase(
    syncEngineDelegate: (any SyncEngineDelegate)? = nil
  ) throws {
    defaultDatabase = try appDatabase()
    defaultSyncEngine = try SyncEngine(
      for: defaultDatabase,
      tables: RemindersList.self,
      RemindersListAsset.self,
      Reminder.self,
      Tag.self,
      ReminderTag.self,
      delegate: syncEngineDelegate
    )
  }
}
```

#### In App Init

```swift
@main
struct MyApp: App {
  @State var syncEngineDelegate = MySyncEngineDelegate()

  init() {
    try! prepareDependencies {
      try $0.bootstrapDatabase(syncEngineDelegate: syncEngineDelegate)
    }
  }

  var body: some Scene {
    WindowGroup {
      ContentView()
    }
  }
}
```

### Sharing Records

#### Share a Record

```swift
@Dependency(\.defaultSyncEngine) var syncEngine

func shareButtonTapped() {
  Task {
    sharedRecord = try await syncEngine.share(record: counter) { share in
      share[CKShare.SystemFieldKey.title] = "Join my counter!"
    }
  }
}
```

#### Present Share Sheet

```swift
struct CounterRow: View {
  let counter: Counter
  @State var sharedRecord: SharedRecord?
  @Dependency(\.defaultSyncEngine) var syncEngine

  var body: some View {
    HStack {
      Text("\(counter.count)")
      Button {
        shareButtonTapped()
      } label: {
        Image(systemName: "square.and.arrow.up")
      }
    }
    .sheet(item: $sharedRecord) { sharedRecord in
      CloudSharingView(sharedRecord: sharedRecord)
    }
  }

  func shareButtonTapped() {
    Task {
      sharedRecord = try await syncEngine.share(record: counter) { share in
        share[CKShare.SystemFieldKey.title] = "Join my counter!"
      }
    }
  }
}
```

### Accepting Shares

#### In SceneDelegate

```swift
class SceneDelegate: UIResponder, UIWindowSceneDelegate {
  @Dependency(\.defaultSyncEngine) var syncEngine
  var window: UIWindow?

  func windowScene(
    _ windowScene: UIWindowScene,
    userDidAcceptCloudKitShareWith cloudKitShareMetadata: CKShare.Metadata
  ) {
    Task {
      try await syncEngine.acceptShare(metadata: cloudKitShareMetadata)
    }
  }

  func scene(
    _ scene: UIScene,
    willConnectTo session: UISceneSession,
    options connectionOptions: UIScene.ConnectionOptions
  ) {
    guard let cloudKitShareMetadata = connectionOptions.cloudKitShareMetadata
    else { return }

    Task {
      try await syncEngine.acceptShare(metadata: cloudKitShareMetadata)
    }
  }
}
```

#### Register SceneDelegate

```swift
class AppDelegate: UIResponder, UIApplicationDelegate {
  func application(
    _ application: UIApplication,
    configurationForConnecting connectingSceneSession: UISceneSession,
    options: UIScene.ConnectionOptions
  ) -> UISceneConfiguration {
    let configuration = UISceneConfiguration(
      name: "Default Configuration",
      sessionRole: connectingSceneSession.role
    )
    configuration.delegateClass = SceneDelegate.self
    return configuration
  }
}

@main
struct MyApp: App {
  @UIApplicationDelegateAdaptor var delegate: AppDelegate
  // ...
}
```

### SyncEngineDelegate

#### Account Change Handling

```swift
@MainActor
@Observable
class MySyncEngineDelegate: SyncEngineDelegate {
  var isDeleteLocalDataAlertPresented = false

  func syncEngine(
    _ syncEngine: SQLiteData.SyncEngine,
    accountChanged changeType: CKSyncEngine.Event.AccountChange.ChangeType
  ) async {
    switch changeType {
    case .signIn:
      // User signed into iCloud
      break
    case .signOut, .switchAccounts:
      // Prompt user to reset local data
      isDeleteLocalDataAlertPresented = true
    @unknown default:
      break
    }
  }
}
```

#### Delete Local Data on Sign Out

```swift
@main
struct MyApp: App {
  @Dependency(\.defaultSyncEngine) var syncEngine
  @State var syncEngineDelegate = MySyncEngineDelegate()

  var body: some Scene {
    WindowGroup {
      ContentView()
        .alert(
          "Reset local data?",
          isPresented: $syncEngineDelegate.isDeleteLocalDataAlertPresented
        ) {
          Button("Reset", role: .destructive) {
            Task {
              try await syncEngine.deleteLocalData()
            }
          }
        } message: {
          Text(
            """
            You are no longer logged into iCloud. Would you like to reset your local data to the \
            defaults? This will not affect your data in iCloud.
            """
          )
        }
    }
  }
}
```

### Querying Sync Metadata

#### Check Share Status

```swift
@FetchAll(
  RemindersList
    .group(by: \.id)
    .leftJoin(SyncMetadata.all) { $0.syncMetadataID.eq($1.id) }
    .select {
      ReminderListState.Columns(
        remindersList: $0,
        share: $1.share
      )
    }
)
var remindersLists

@Selection
struct ReminderListState {
  var remindersList: RemindersList
  @Column(as: CKShare?.self)
  var share: CKShare?
}
```

#### Display Share Status

```swift
if let share = reminderListState.share {
  if share.currentUserParticipant?.role == .owner {
    Text("Shared by you")
  } else {
    Text("Shared with you")
  }
}
```

### Database Configuration for Sync

```swift
var configuration = Configuration()
configuration.foreignKeysEnabled = true
configuration.prepareDatabase { db in
  try db.attachMetadatabase()  // Required for CloudKit sync
}

let database = try SQLiteData.defaultDatabase(configuration: configuration)
```

### Best Practices

1. **Always use `try db.attachMetadatabase()`** in database configuration
2. **Register all synced tables** in `SyncEngine` init
3. **Handle account changes** with `SyncEngineDelegate`
4. **Prompt before `deleteLocalData()`** - it's destructive
5. **Use `@Column(as: CKShare?.self)`** for share status in queries
6. **Configure `SceneDelegate`** for accepting shares
7. **Foreign keys must be enabled** for proper sync relationships

---

## Dependency Injection

Patterns for integrating sqlite-data with swift-dependencies for dependency injection.

### Accessing Dependencies

#### @Dependency in Views

```swift
struct CountersListView: View {
  @FetchAll var counters: [Counter]
  @Dependency(\.defaultDatabase) var database

  var body: some View {
    List {
      ForEach(counters) { counter in
        CounterRow(counter: counter)
      }
    }
    .toolbar {
      Button("Add") {
        withErrorReporting {
          try database.write { db in
            try Counter.insert { Counter.Draft() }.execute(db)
          }
        }
      }
    }
  }
}
```

#### @Dependency in @Observable Models

```swift
@Observable
@MainActor
class RemindersListsModel {
  @ObservationIgnored
  @FetchAll(RemindersList.all)
  var remindersLists

  @ObservationIgnored
  @Dependency(\.defaultDatabase) private var database

  @ObservationIgnored
  @Dependency(\.defaultSyncEngine) var syncEngine

  func addList() {
    withErrorReporting {
      try database.write { db in
        try RemindersList.insert { RemindersList.Draft() }.execute(db)
      }
    }
  }
}
```

#### In TCA Reducers

```swift
@Reducer
struct CountersListFeature {
  struct State {
    // ...
  }
  enum Action {
    // ...
  }

  @Dependency(\.defaultDatabase) var database
  @Dependency(\.defaultSyncEngine) var syncEngine

  var body: some ReducerOf<Self> {
    Reduce { state, action in
      switch action {
      case .addCounter:
        return .run { send in
          try await database.write { db in
            try Counter.insert { Counter.Draft() }.execute(db)
          }
        }
      }
    }
  }
}
```

### Bootstrap Database

#### Extension on DependencyValues

```swift
extension DependencyValues {
  mutating func bootstrapDatabase(
    syncEngineDelegate: (any SyncEngineDelegate)? = nil
  ) throws {
    defaultDatabase = try appDatabase()
    defaultSyncEngine = try SyncEngine(
      for: defaultDatabase,
      tables: RemindersList.self,
      RemindersListAsset.self,
      Reminder.self,
      Tag.self,
      ReminderTag.self,
      delegate: syncEngineDelegate
    )
  }
}
```

#### Call in App Init

```swift
@main
struct MyApp: App {
  @State var syncEngineDelegate = MySyncEngineDelegate()

  init() {
    try! prepareDependencies {
      try $0.bootstrapDatabase(syncEngineDelegate: syncEngineDelegate)
    }
  }

  var body: some Scene {
    WindowGroup {
      ContentView()
    }
  }
}
```

### Preview Dependencies

#### Override for Previews

```swift
#Preview {
  let _ = prepareDependencies {
    $0.defaultDatabase = .swiftUIDatabase
  }

  NavigationStack {
    CaseStudyView {
      SwiftUIDemo()
    }
  }
}
```

#### Preview Database

```swift
extension DatabaseWriter where Self == DatabaseQueue {
  static var swiftUIDatabase: Self {
    let databaseQueue = try! DatabaseQueue()
    var migrator = DatabaseMigrator()
    migrator.registerMigration("Create 'facts' table") { db in
      try #sql(
        """
        CREATE TABLE "facts" (
          "id" INTEGER PRIMARY KEY AUTOINCREMENT,
          "body" TEXT NOT NULL
        ) STRICT
        """
      ).execute(db)
    }
    try! migrator.migrate(databaseQueue)
    return databaseQueue
  }
}
```

### withDependencies for Child Models

When creating child models that need access to dependencies:

```swift
@Observable
class ParentModel {
  @ObservationIgnored
  @Dependency(\.defaultDatabase) var database

  func createChildModel() -> ChildModel {
    withDependencies(from: self) {
      ChildModel()
    }
  }
}
```

#### In TCA

```swift
@Reducer
struct ParentFeature {
  @Dependency(\.defaultDatabase) var database

  var body: some ReducerOf<Self> {
    Reduce { state, action in
      switch action {
      case .addButtonTapped:
        state.destination = .form(
          withDependencies(from: self) {
            FormFeature.State()
          }
        )
        return .none
      }
    }
  }
}
```

### Accessing Other Dependencies

#### Date Dependency

```swift
nonisolated extension Reminder.TableColumns {
  var isPastDue: some QueryExpression<Bool> {
    @Dependency(\.date.now) var now
    return !isCompleted && #sql("coalesce(date(\(dueDate)) < date(\(now)), 0)")
  }
}
```

#### UUID Dependency

```swift
@Dependency(\.uuid) var uuid

func createNewItem() {
  let id = uuid()
  // Use id
}
```

#### Context Dependency

```swift
@Dependency(\.context) var context

switch context {
case .live:
  // Production behavior
case .preview:
  // Preview behavior
case .test:
  // Test behavior
}
```

### Database Writer Protocol

The `defaultDatabase` dependency conforms to `DatabaseWriter`:

```swift
protocol DatabaseWriter {
  func read<T>(_ block: (Database) throws -> T) throws -> T
  func write<T>(_ block: (Database) throws -> T) throws -> T
}
```

#### Read Transaction

```swift
@Dependency(\.defaultDatabase) var database

let counters = try database.read { db in
  try Counter.order(by: \.id).fetchAll(db)
}
```

#### Write Transaction

```swift
@Dependency(\.defaultDatabase) var database

try database.write { db in
  try Counter.insert { Counter.Draft() }.execute(db)
}
```

#### Async Write

```swift
try await database.write { db in
  try Fact.insert { Fact.Draft(body: fact) }.execute(db)
}
```

### Best Practices

1. **Use `@ObservationIgnored`** on `@Dependency` in `@Observable` classes
2. **Call `prepareDependencies`** in App init, not in previews when possible
3. **Use `withDependencies(from:)`** when creating child models
4. **Bootstrap database once** at app launch
5. **Override dependencies** in previews and tests
6. **Use `.defaultDatabase`** for all database access
7. **Mark database dependency as private** when only used internally

---

## Testing

Patterns for testing database code with swift-testing and swift-dependencies.

### Test Suite Setup

#### Basic Test Suite

```swift
@Suite(
  .dependencies {
    try $0.bootstrapDatabase()
  }
)
struct MyTestSuite {}
```

#### With Controlled Dependencies

```swift
@Suite(
  .dependency(\.continuousClock, ImmediateClock()),
  .dependency(\.date.now, Date(timeIntervalSince1970: 1_234_567_890)),
  .dependency(\.uuid, .incrementing),
  .dependencies {
    try $0.bootstrapDatabase()
    try await $0.defaultSyncEngine.sendChanges()
  },
  .snapshots(record: .failed)
)
struct BaseTestSuite {}
```

#### Nested Test Suites

```swift
extension BaseTestSuite {
  @MainActor
  struct RemindersDetailsTests {
    @Dependency(\.defaultDatabase) var database

    @Test func basics() async throws {
      // Test implementation
    }
  }
}
```

### Reading from Database

#### Fetch for Assertions

```swift
@Test func testCounter() async throws {
  @Dependency(\.defaultDatabase) var database

  // Act
  try database.write { db in
    try Counter.insert { Counter.Draft(count: 5) }.execute(db)
  }

  // Assert
  let counter = try database.read { db in
    try Counter.fetchOne(db)!
  }
  #expect(counter.count == 5)
}
```

#### Fetch One Record

```swift
let remindersList = try await database.read { try RemindersList.fetchOne($0)! }
```

#### Fetch All Records

```swift
let attendees = try database.read { db in
  try Attendee.where { $0.syncUpID.eq(syncUp.id) }.fetchAll(db)
}
```

### Seeding Test Data

#### Define Seed Extension

```swift
extension Database {
  func seed() throws {
    try seed {
      SyncUp(id: UUID(1), seconds: 60, theme: .appOrange, title: "Design")
      SyncUp(id: UUID(2), seconds: 60 * 10, theme: .periwinkle, title: "Engineering")

      for name in ["Blob", "Blob Jr", "Blob Sr"] {
        Attendee.Draft(name: name, syncUpID: UUID(1))
      }
      for name in ["Blob", "Blob Jr"] {
        Attendee.Draft(name: name, syncUpID: UUID(2))
      }

      Meeting.Draft(
        date: Date().addingTimeInterval(-60 * 60 * 24 * 7),
        syncUpID: UUID(1),
        transcript: "Meeting notes..."
      )
    }
  }
}
```

#### Use in Test Suite

```swift
@Suite(
  .dependencies {
    try $0.bootstrapDatabase()
    try $0.defaultDatabase.write { db in
      try db.seed()
    }
    $0.uuid = .incrementing
  }
)
struct SyncUpFormTests {}
```

### Testing with @Fetch

#### Load and Assert

```swift
@Test func testRemindersDetail() async throws {
  @Dependency(\.defaultDatabase) var database

  let remindersList = try await database.read { try RemindersList.fetchOne($0)! }
  let model = RemindersDetailModel(detailType: .remindersList(remindersList))

  // Load the @Fetch query
  try await model.$reminderRows.load()

  // Assert on results
  #expect(model.reminderRows.count == 4)
}
```

### Snapshot Testing

#### Inline Snapshots

```swift
@Test func testModel() async throws {
  let model = RemindersDetailModel(detailType: .remindersList(remindersList))
  try await model.$reminderRows.load()

  assertInlineSnapshot(of: model.reminderRows, as: .customDump) {
    #"""
    [
      [0]: RemindersDetailModel.Row(
        reminder: Reminder(
          id: UUID(00000000-0000-0000-0000-000000000004),
          title: "Haircut",
          dueDate: Date(2009-02-11T23:31:30.000Z),
          status: .incomplete
        )
      )
    ]
    """#
  }
}
```

#### CustomDumpReflectable for Consistent Output

Handle types that don't dump consistently (like SwiftUI.Color):

```swift
extension RemindersList: @retroactive CustomDumpReflectable {
  public var customDumpMirror: Mirror {
    Mirror(
      self,
      children: [
        "id": id,
        "color": Color.HexRepresentation(queryOutput: color).hexValue ?? 0,
        "position": position,
        "title": title,
      ],
      displayStyle: .struct
    )
  }
}
```

### Testing Writes

#### Test Insert

```swift
@Test func testInsert() async throws {
  @Dependency(\.defaultDatabase) var database

  try database.write { db in
    try Counter.insert { Counter.Draft(count: 42) }.execute(db)
  }

  let counters = try database.read { db in
    try Counter.fetchAll(db)
  }

  #expect(counters.count == 1)
  #expect(counters[0].count == 42)
}
```

#### Test Update

```swift
@Test func testUpdate() async throws {
  @Dependency(\.defaultDatabase) var database

  let id = UUID()
  try database.write { db in
    try Counter.insert { Counter.Draft(id: id, count: 0) }.execute(db)
    try Counter.find(id).update { $0.count += 1 }.execute(db)
  }

  let counter = try database.read { db in
    try Counter.find(id).fetchOne(db)!
  }

  #expect(counter.count == 1)
}
```

#### Test Delete

```swift
@Test func testDelete() async throws {
  @Dependency(\.defaultDatabase) var database

  let id = UUID()
  try database.write { db in
    try Counter.insert { Counter.Draft(id: id) }.execute(db)
    try Counter.find(id).delete().execute(db)
  }

  let counters = try database.read { db in
    try Counter.fetchAll(db)
  }

  #expect(counters.isEmpty)
}
```

### Controlled Dependencies

#### UUID Incrementing

```swift
@Suite(.dependency(\.uuid, .incrementing))
struct MyTests {
  @Test func testUUIDs() {
    @Dependency(\.uuid) var uuid
    #expect(uuid() == UUID(0))
    #expect(uuid() == UUID(1))
    #expect(uuid() == UUID(2))
  }
}
```

#### Fixed Date

```swift
@Suite(.dependency(\.date.now, Date(timeIntervalSince1970: 1_234_567_890)))
struct DateTests {
  @Test func testDateComparison() {
    @Dependency(\.date.now) var now
    // now is always Date(timeIntervalSince1970: 1_234_567_890)
  }
}
```

#### Immediate Clock

```swift
@Suite(.dependency(\.continuousClock, ImmediateClock()))
struct TimerTests {
  @Test func testTimer() async {
    // All sleep operations complete immediately
  }
}
```

### Best Practices

1. **Use `@Suite(.dependencies {})`** to configure test dependencies
2. **Seed data in suite setup** for consistent test state
3. **Use `.incrementing` UUID** for predictable IDs in tests
4. **Use fixed `date.now`** for time-dependent tests
5. **Use `ImmediateClock`** to avoid waiting in tests
6. **Load `@Fetch` queries** with `$property.load()` before assertions
7. **Use `assertInlineSnapshot`** for comprehensive output validation
8. **Define `CustomDumpReflectable`** for types with inconsistent dumps
9. **Bootstrap database** in test suite setup
10. **Test in transactions** - each test gets fresh database state

---

## Advanced Query Features

Database triggers, custom functions, and full-text search with FTS5.

### Database Triggers

#### Temporary Triggers

Temporary triggers are created in memory and don't persist to disk:

```swift
try database.write { db in
  try RemindersList.createTemporaryTrigger(
    after: .insert { new in
      RemindersList
        .find(new.id)
        .update {
          $0.position = RemindersList.select { ($0.position.max() ?? -1) + 1 }
        }
    }
  ).execute(db)
}
```

#### Auto-Increment Position

```swift
try Reminder.createTemporaryTrigger(
  after: .insert { new in
    Reminder
      .find(new.id)
      .update {
        $0.position = Reminder.select { ($0.position.max() ?? -1) + 1 }
      }
  }
).execute(db)
```

#### Conditional Triggers

Execute trigger only when condition is met:

```swift
try RemindersList.createTemporaryTrigger(
  after: .delete { _ in
    Values($createDefaultRemindersList())
  } when: { _ in
    !RemindersList.exists()
  }
).execute(db)
```

#### FTS5 Sync Trigger

Keep FTS5 table in sync with main table:

```swift
try Reminder.createTemporaryTrigger(
  after: .insert { new in
    ReminderText.insert {
      ReminderText.Columns(
        rowid: new.rowid,
        title: new.title,
        notes: new.notes.replace("\n", " "),
        tags: ""
      )
    }
  }
).execute(db)

try Reminder.createTemporaryTrigger(
  after: .update { new in
    ReminderText.find(new.rowid).update {
      $0.title = new.title
      $0.notes = new.notes.replace("\n", " ")
    }
  }
).execute(db)

try Reminder.createTemporaryTrigger(
  after: .delete { old in
    ReminderText.find(old.rowid).delete()
  }
).execute(db)
```

### Database Functions

#### Define Custom Function

Use `@DatabaseFunction` to create Swift functions callable from SQL:

```swift
@DatabaseFunction
nonisolated func createDefaultRemindersList() {
  Task {
    @Dependency(\.defaultDatabase) var database
    try await database.write { db in
      try RemindersList.insert {
        RemindersList.Draft(
          title: RemindersList.defaultTitle,
          color: RemindersList.defaultColor
        )
      }.execute(db)
    }
  }
}
```

#### Register Function

```swift
configuration.prepareDatabase { db in
  db.add(function: $createDefaultRemindersList)
  db.add(function: $handleReminderStatusUpdate)
}
```

#### Use in Triggers

```swift
try RemindersList.createTemporaryTrigger(
  after: .delete { _ in
    Values($createDefaultRemindersList())
  } when: { _ in
    !RemindersList.exists()
  }
).execute(db)
```

#### Async Database Function

```swift
@DatabaseFunction
nonisolated func handleReminderStatusUpdate() {
  reminderStatusMutex.withLock {
    $0?.cancel()
    $0 = Task {
      @Dependency(\.defaultDatabase) var database
      try await Task.sleep(for: .seconds(0.4))
      try await database.write { db in
        try Reminder
          .where { $0.status.eq(.completing) }
          .update { $0.status = #bind(.completed) }
          .execute(db)
      }
    }
  }
}
```

### Full-Text Search (FTS5)

#### Define FTS5 Table

```swift
@Table
struct ReminderText: FTS5 {
  let rowid: Int
  let title: String
  let notes: String
  let tags: String
}
```

#### Create FTS5 Virtual Table

```swift
migrator.registerMigration("Create FTS5 table") { db in
  try #sql(
    """
    CREATE VIRTUAL TABLE "reminderTexts" USING fts5(
      "title",
      "notes",
      "tags",
      tokenize = 'trigram'
    )
    """
  ).execute(db)
}
```

#### Search with FTS5

```swift
func baseQuery(searchText: String, searchTokens: [Token]) -> some Query {
  ReminderText
    .where {
      if !searchText.isEmpty || !searchTokens.isEmpty {
        $0.match(buildFTS5Query(searchText: searchText, tokens: searchTokens))
      }
    }
    .join(Reminder.all) { $0.rowid.eq($1.rowid) }
}
```

#### FTS5 Highlight

Highlight matching text:

```swift
ReminderText.select {
  SearchRow.Columns(
    title: $0.title.highlight("**", "**"),
    tags: $0.tags.highlight("**", "**")
  )
}
```

#### FTS5 Snippet

Extract relevant snippets with context:

```swift
ReminderText.select {
  SearchRow.Columns(
    notes: $0.notes.snippet("**", "**", "...", 64).replace("\n", " ")
  )
}
```

**Parameters:**
- First arg: Start marker for matches
- Second arg: End marker for matches
- Third arg: Ellipsis for truncated text
- Fourth arg: Maximum tokens (not characters)

#### FTS5 Tokenizers

Common tokenizers:

```sql
-- Trigram (for autocomplete, partial matching)
tokenize = 'trigram'

-- Porter stemming (for word variants)
tokenize = 'porter'

-- Unicode61 (default, case-insensitive)
tokenize = 'unicode61'
```

#### Complex FTS5 Queries

```swift
func buildFTS5Query(searchText: String, tokens: [Token]) -> String {
  var components: [String] = []

  if !searchText.isEmpty {
    components.append(searchText)
  }

  for token in tokens {
    switch token.kind {
    case .tag:
      components.append("\(token.rawValue)*")
    case .near:
      components.append("NEAR(\(token.rawValue), 10)")
    }
  }

  return components.joined(separator: " ")
}
```

### Best Practices

1. **Use temporary triggers** for app-specific logic (don't persist to schema)
2. **Sync FTS5 with triggers** to keep search index up-to-date
3. **Use FTS5 for search** instead of LIKE queries
4. **Use `.highlight()` and `.snippet()`** for better search UX
5. **Use trigram tokenizer** for autocomplete-style search

---

## Advanced Optimization & Aggregation

Performance optimization, custom aggregates, JSON aggregation, and self-joins.

### Performance Optimization

#### Indexes on Foreign Keys

```swift
migrator.registerMigration("Create foreign key indexes") { db in
  try #sql(
    """
    CREATE INDEX IF NOT EXISTS "idx_reminders_remindersListID"
    ON "reminders"("remindersListID")
    """
  ).execute(db)

  try #sql(
    """
    CREATE INDEX IF NOT EXISTS "idx_remindersTags_reminderID"
    ON "remindersTags"("reminderID")
    """
  ).execute(db)
}
```

#### Query Profiling

Enable in DEBUG builds:

```swift
#if DEBUG
  configuration.prepareDatabase { db in
    db.trace(options: .profile) {
      logger.debug("\($0.expandedDescription)")
    }
  }
#endif
```

#### Batch Operations

Perform multiple operations in single transaction:

```swift
try database.write { db in
  // All succeed or all fail together
  try Counter.insert { Counter.Draft() }.execute(db)
  try Counter.find(otherID).delete().execute(db)
  try Counter.find(thirdID).update { $0.count = 0 }.execute(db)
}
```

### Custom Aggregate Functions

Define complex aggregation logic in Swift with `@DatabaseFunction`:

```swift
@DatabaseFunction
func mode(priority priorities: some Sequence<Reminder.Priority?>) -> Reminder.Priority? {
    var occurrences: [Reminder.Priority: Int] = [:]
    for priority in priorities {
        guard let priority else { continue }
        occurrences[priority, default: 0] += 1
    }
    return occurrences.max { $0.value < $1.value }?.key
}

// Register in configuration
configuration.prepareDatabase { db in
    db.add(function: $mode)
}

// Use in queries
let results = try RemindersList
    .group(by: \.id)
    .leftJoin(Reminder.all) { $0.id.eq($1.remindersListID) }
    .select { ($0.title, $mode(priority: $1.priority)) }
    .fetchAll(db)
```

### JSON Aggregation

Build JSON arrays directly in queries:

```swift
// Aggregate rows into JSON array
let storesWithItems = try Store
    .group(by: \.id)
    .leftJoin(Item.all) { $0.id.eq($1.storeID) }
    .select {
        (
            $0.name,
            $1.title.jsonGroupArray()  // ["item1", "item2", ...]
        )
    }
    .fetchAll(db)

// With filtering
let activeItemsJson = try Store
    .group(by: \.id)
    .leftJoin(Item.all) { $0.id.eq($1.storeID) }
    .select {
        $1.title.jsonGroupArray(filter: $1.isActive)
    }
    .fetchAll(db)
```

### String Aggregation

Concatenate values from multiple rows:

```swift
let itemsWithTags = try Item
    .group(by: \.id)
    .leftJoin(ItemTag.all) { $0.id.eq($1.itemID) }
    .leftJoin(Tag.all) { $1.tagID.eq($2.id) }
    .select {
        (
            $0.title,
            $2.name.groupConcat(separator: ", ")
        )
    }
    .fetchAll(db)
// ("iPhone", "electronics, mobile, apple")
```

### Self-Joins with TableAlias

Query the same table twice (e.g., employee/manager):

```swift
struct ManagerAlias: TableAlias {
    typealias Table = Employee
}

let employeesWithManagers = try Employee
    .leftJoin(Employee.all.as(ManagerAlias.self)) { $0.managerID.eq($1.id) }
    .select {
        (
            employeeName: $0.name,
            managerName: $1.name
        )
    }
    .fetchAll(db)

// Find employees who manage others
let managers = try Employee
    .join(Employee.all.as(ManagerAlias.self)) { $0.id.eq($1.managerID) }
    .select { $0 }
    .distinct()
    .fetchAll(db)
```

### Best Practices

1. **Add foreign key indexes** for better join performance
2. **Profile queries in DEBUG** to identify slow operations
3. **Batch operations** in single transaction for consistency
4. **Use custom aggregates** for mode, median, or complex statistics
5. **Use TableAlias** for self-referential joins (org charts, trees)

---

## Schema Composition

Patterns for reusable column groups, single-table inheritance, and database views.

### @Selection Column Groups

Group related columns into reusable types:

```swift
@Selection
struct Timestamps {
    let createdAt: Date
    let updatedAt: Date?
}

@Table
nonisolated struct RemindersList: Identifiable {
    let id: UUID
    var title = ""
    let timestamps: Timestamps  // Embedded column group
}
```

**SQL flattens all groups** - no nested structure in database:

```sql
CREATE TABLE "remindersLists" (
    "id" TEXT PRIMARY KEY NOT NULL DEFAULT (uuid()),
    "title" TEXT NOT NULL DEFAULT '',
    "createdAt" TEXT NOT NULL,
    "updatedAt" TEXT
) STRICT
```

#### Querying Column Groups

```swift
// Access fields with dot syntax
RemindersList.where { $0.timestamps.createdAt <= cutoffDate }.fetchAll(db)

// Nest groups in @Selection results
@Selection
struct Row {
    let reminderTitle: String
    let timestamps: Timestamps
}

Reminder.join(RemindersList.all) { $0.remindersListID.eq($1.id) }
    .select { Row.Columns(reminderTitle: $0.title, timestamps: $0.timestamps) }
```

### Single-Table Inheritance

Model polymorphic data using `@CasePathable @Selection` enums:

```swift
import CasePaths

@Table
nonisolated struct Attachment: Identifiable {
    let id: UUID
    let kind: Kind

    @CasePathable @Selection
    enum Kind {
        case link(URL)
        case note(String)
        case image(URL)
    }
}
```

**SQL flattens all cases into nullable columns:**

```sql
CREATE TABLE "attachments" (
    "id" TEXT PRIMARY KEY NOT NULL DEFAULT (uuid()),
    "link" TEXT, "note" TEXT, "image" TEXT
) STRICT
```

#### Querying Enum Tables

```swift
// Filter by case
let images = try Attachment.where { $0.kind.image.isNot(nil) }.fetchAll(db)

// Insert with specific case
try Attachment.insert { Attachment.Draft(kind: .note("Hello!")) }.execute(db)
// Inserts: (id, NULL, 'Hello!', NULL)

// Update changes which columns are populated
try Attachment.find(id).update {
    $0.kind = .link(URL(string: "https://example.com")!)
}.execute(db)
// Sets link, NULLs note and image
```

### Database Views

Create temporary views for complex queries using `@Table @Selection`:

```swift
@Table @Selection
private struct ReminderWithList {
    let reminderTitle: String
    let remindersListTitle: String
}

try database.write { db in
    try ReminderWithList.createTemporaryView(
        as: Reminder
            .join(RemindersList.all) { $0.remindersListID.eq($1.id) }
            .select {
                ReminderWithList.Columns(
                    reminderTitle: $0.title,
                    remindersListTitle: $1.title
                )
            }
    ).execute(db)
}

// Query like any table - join complexity hidden
let results = try ReminderWithList
    .order { ($0.remindersListTitle, $0.reminderTitle) }
    .fetchAll(db)
```

#### Updatable Views

Enable inserts/updates with `INSTEAD OF` triggers:

```swift
try ReminderWithList.createTemporaryTrigger(
    insteadOf: .insert { new in
        Reminder.insert { ($0.title, $0.remindersListID) }
            values: { (new.reminderTitle, RemindersList.select(\.id)
                .where { $0.title.eq(new.remindersListTitle) }) }
    }
).execute(db)
```

### When to Use Each Pattern

| Pattern | Use Case |
|---------|----------|
| `@Selection` groups | Reuse timestamp/audit columns across tables |
| `@CasePathable` enum | Polymorphic types (attachments, content blocks) |
| `@Table @Selection` view | Hide join complexity, create reusable queries |
| Temporary view | Query varies by runtime |
| Permanent view | Query used across restarts, rarely changes |

---
