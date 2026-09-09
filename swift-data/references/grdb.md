# GRDB

Full content of the former standalone `grdb` skill. Direct SQLite access using [GRDB.swift](https://github.com/groue/GRDB.swift) — a type-safe Swift wrapper with full SQLite power when you need it (raw SQL, complex joins, window functions, reactive queries).

## Section guide

**Prefer reading a section over skipping it if there's even a small chance the content may be required** — it's better to have the context than to miss a pattern or make a mistake.

| Section | Read When |
|-----------|-----------|
| [Getting Started with GRDB](#getting-started-with-grdb) | Setting up DatabaseQueue or DatabasePool |
| [GRDB Queries](#grdb-queries) | Writing raw SQL, Record types, type-safe queries |
| [ValueObservation - Reactive Queries](#valueobservation---reactive-queries) | Reactive queries, SwiftUI integration |
| [GRDB Migrations](#grdb-migrations) | DatabaseMigrator, schema evolution |
| [GRDB Performance](#grdb-performance) | EXPLAIN QUERY PLAN, indexing, profiling |

## When to Use GRDB vs SQLiteData

| Scenario | Use |
|----------|-----|
| Type-safe @Table models | SQLiteData |
| CloudKit sync needed | SQLiteData |
| Complex joins (4+ tables) | GRDB |
| Window functions (ROW_NUMBER, RANK) | GRDB |
| Performance-critical raw SQL | GRDB |
| Reactive queries (ValueObservation) | GRDB |

See `sqlitedata.md` for the SQLiteData side of this comparison.

## Core Workflow

1. Choose DatabaseQueue (single connection) or DatabasePool (concurrent reads)
2. Define migrations with DatabaseMigrator
3. Create Record types (Codable, FetchableRecord, PersistableRecord)
4. Write queries with raw SQL or QueryInterface
5. Use ValueObservation for reactive updates

## Requirements

- iOS 13+, macOS 10.15+
- Swift 5.7+
- GRDB.swift 6.0+

## Common Mistakes

1. **Performance assumptions without EXPLAIN PLAN** — Assuming your query is fast or slow without checking `EXPLAIN QUERY PLAN` is guessing. Always profile queries with EXPLAIN before optimizing.

2. **Missing indexes on WHERE clauses** — Queries filtering on non-indexed columns scan the entire table. Index any column used in WHERE, JOIN, or ORDER BY clauses for large tables.

3. **Improper migration ordering** — Running migrations out of order or skipping intermediate versions breaks schema consistency. Always apply migrations sequentially; never jump versions.

4. **Record conformance shortcuts** — Not conforming Record types to `PersistableRecord` or `FetchableRecord` correctly leads to silent data loss or deserialization failures. Always implement all required protocols correctly.

5. **ValueObservation without proper cleanup** — Forgetting to cancel ValueObservation when views disappear causes memory leaks and stale data subscriptions. Store the cancellable and clean up in deinit.

---


## Getting Started with GRDB

### Installation

Add GRDB to your Swift Package Manager dependencies:

```swift
dependencies: [
    .package(url: "https://github.com/groue/GRDB.swift", from: "6.0.0")
]
```

### DatabaseQueue (Single Connection)

Use DatabaseQueue for most apps - simpler and sufficient for typical usage patterns.

```swift
import GRDB

// File-based database
let dbPath = NSSearchPathForDirectoriesInDomains(
    .documentDirectory, .userDomainMask, true
)[0]
let dbQueue = try DatabaseQueue(path: "\(dbPath)/app.sqlite")

// In-memory database (useful for tests)
let dbQueue = try DatabaseQueue()
```

#### Basic Read/Write Pattern

```swift
// Reading data
let tracks = try dbQueue.read { db in
    try Track.fetchAll(db)
}

// Writing data
try dbQueue.write { db in
    try track.insert(db)
}
```

### DatabasePool (Connection Pool)

Use DatabasePool for apps with heavy concurrent access - allows concurrent reads while writing.

```swift
import GRDB

let dbPool = try DatabasePool(path: "\(dbPath)/app.sqlite")

// Concurrent reads
let result1 = try dbPool.read { db in try Track.fetchAll(db) }
let result2 = try dbPool.read { db in try Album.fetchAll(db) }

// Exclusive writes
try dbPool.write { db in
    try track.insert(db)
}
```

#### When to Use Each

| Use Case | Choice |
|----------|--------|
| Most apps | DatabaseQueue |
| Heavy concurrent writes from multiple threads | DatabasePool |
| Unit tests | DatabaseQueue (in-memory) |
| Background sync with UI updates | DatabasePool |

### Database Configuration

```swift
var config = Configuration()

// Enable tracing for debugging
config.trace = { print($0) }

// Enable foreign key support
config.prepareDatabase { db in
    try db.execute(sql: "PRAGMA foreign_keys = ON")
}

let dbQueue = try DatabaseQueue(path: dbPath, configuration: config)
```

### Async/Await Support

```swift
// Async read
let tracks = try await dbQueue.read { db in
    try Track.fetchAll(db)
}

// Async write
try await dbQueue.write { db in
    try track.insert(db)
}
```

### App Lifecycle Pattern

```swift
final class DatabaseManager {
    static let shared = DatabaseManager()

    let dbQueue: DatabaseQueue

    private init() {
        let path = NSSearchPathForDirectoriesInDomains(
            .documentDirectory, .userDomainMask, true
        )[0] + "/app.sqlite"

        do {
            var migrator = DatabaseMigrator()
            // Register migrations...

            dbQueue = try DatabaseQueue(path: path)
            try migrator.migrate(dbQueue)
        } catch {
            fatalError("Database setup failed: \(error)")
        }
    }
}
```

### Dropping Down from SQLiteData

When using SQLiteData but need GRDB for specific operations:

```swift
import SQLiteData
import GRDB

@Dependency(\.database) var database  // SQLiteData Database

// Access underlying GRDB DatabaseQueue
try await database.database.write { db in
    // Full GRDB power here
    try db.execute(sql: "CREATE INDEX idx_genre ON tracks(genre)")
}
```

Common scenarios for dropping down:
- Complex JOIN queries across 4+ tables
- Window functions (ROW_NUMBER, RANK, LAG/LEAD)
- Custom migrations with data transforms
- Bulk SQL operations
- ValueObservation setup

---

## GRDB Queries

### Raw SQL Queries

```swift
// Fetch all rows
let rows = try dbQueue.read { db in
    try Row.fetchAll(db, sql: "SELECT * FROM tracks WHERE genre = ?", arguments: ["Rock"])
}

// Access row values
for row in rows {
    let title: String = row["title"]
    let duration: Double = row["duration"]
}

// Fetch single value
let count = try dbQueue.read { db in
    try Int.fetchOne(db, sql: "SELECT COUNT(*) FROM tracks")
}

// Write data
try dbQueue.write { db in
    try db.execute(sql: "INSERT INTO tracks (id, title) VALUES (?, ?)",
                   arguments: ["1", "Song"])
}
```

### Record Types

#### Codable + PersistableRecord

```swift
struct Track: Codable, PersistableRecord {
    var id: String
    var title: String
    var artist: String

    static let databaseTableName = "tracks"
}

// Insert/Update/Delete
try dbQueue.write { db in
    try track.insert(db)
    try track.update(db)
    try track.delete(db)
}
```

#### FetchableRecord (Custom Query Results)

```swift
struct TrackInfo: FetchableRecord {
    var title: String
    var albumTitle: String

    init(row: Row) {
        title = row["title"]
        albumTitle = row["album_title"]
    }
}

let results = try dbQueue.read { db in
    try TrackInfo.fetchAll(db, sql: """
        SELECT tracks.title, albums.title as album_title
        FROM tracks JOIN albums ON tracks.albumId = albums.id
        """)
}
```

### Type-Safe Query Interface

```swift
let tracks = try dbQueue.read { db in
    try Track
        .filter(Column("genre") == "Rock")
        .filter(Column("duration") > 180)
        .order(Column("title").asc)
        .limit(10)
        .fetchAll(db)
}
```

### Complex Joins (4+ Tables)

```swift
let sql = """
    SELECT
        tracks.title as track_title,
        albums.title as album_title,
        artists.name as artist_name,
        COUNT(plays.id) as play_count
    FROM tracks
    JOIN albums ON tracks.albumId = albums.id
    JOIN artists ON albums.artistId = artists.id
    LEFT JOIN plays ON plays.trackId = tracks.id
    WHERE artists.genre = ?
    GROUP BY tracks.id
    HAVING play_count > 10
    ORDER BY play_count DESC
    """

struct TrackStats: FetchableRecord {
    var trackTitle, albumTitle, artistName: String
    var playCount: Int

    init(row: Row) {
        trackTitle = row["track_title"]
        albumTitle = row["album_title"]
        artistName = row["artist_name"]
        playCount = row["play_count"]
    }
}

let stats = try dbQueue.read { db in
    try TrackStats.fetchAll(db, sql: sql, arguments: ["Rock"])
}
```

### Window Functions

```swift
let sql = """
    SELECT title, artist,
        ROW_NUMBER() OVER (PARTITION BY artist ORDER BY duration DESC) as rank,
        LAG(title) OVER (ORDER BY created_at) as previous_track
    FROM tracks
    """

struct RankedTrack: FetchableRecord {
    var title, artist: String
    var rank: Int
    var previousTrack: String?

    init(row: Row) {
        title = row["title"]
        artist = row["artist"]
        rank = row["rank"]
        previousTrack = row["previous_track"]
    }
}
```

### Transactions

```swift
try dbQueue.write { db in
    // Automatic transaction - all or nothing
    for track in tracks {
        try track.insert(db)
    }
}

// Nested with savepoints
try dbQueue.write { db in
    try db.inSavepoint {
        try riskyOperation(db)
        return .commit  // or .rollback
    }
}
```

---

## ValueObservation - Reactive Queries

ValueObservation automatically re-executes queries when database changes occur.

### Basic Observation

```swift
import GRDB
import Combine

let observation = ValueObservation.tracking { db in
    try Track.fetchAll(db)
}

let cancellable = observation.publisher(in: dbQueue)
    .sink(
        receiveCompletion: { _ in },
        receiveValue: { tracks in
            print("Tracks updated: \(tracks.count)")
        }
    )
```

### Filtered Observation

```swift
func observeGenre(_ genre: String) -> ValueObservation<[Track]> {
    ValueObservation.tracking { db in
        try Track.filter(Column("genre") == genre).fetchAll(db)
    }
}
```

### SwiftUI Integration (GRDBQuery)

```swift
import GRDBQuery

struct TracksRequest: Queryable {
    static var defaultValue: [Track] { [] }

    func publisher(in dbQueue: DatabaseQueue) -> AnyPublisher<[Track], Error> {
        ValueObservation
            .tracking { db in try Track.fetchAll(db) }
            .publisher(in: dbQueue)
            .eraseToAnyPublisher()
    }
}

struct TrackListView: View {
    @Query(TracksRequest())
    var tracks: [Track]

    var body: some View {
        List(tracks) { track in Text(track.title) }
    }
}
```

### Manual Observation (Non-Combine)

```swift
let observer = observation.start(
    in: dbQueue,
    onError: { error in print("Error: \(error)") },
    onChange: { tracks in print("Changed: \(tracks.count)") }
)

observer.cancel()  // When done
```

### Performance Optimization

#### Problem: Re-evaluates on ANY write

```swift
ValueObservation.tracking { db in
    try Track.fetchAll(db)  // CPU spike on unrelated changes
}
```

#### Solution 1: Remove Duplicates

```swift
observation.removeDuplicates().publisher(in: dbQueue)
```

#### Solution 2: Debounce

```swift
observation.removeDuplicates()
    .publisher(in: dbQueue)
    .debounce(for: 0.5, scheduler: DispatchQueue.main)
```

#### Solution 3: Region Tracking

```swift
ValueObservation.tracking(region: Track.all()) { db in
    try Track.fetchAll(db)  // Only Track changes trigger
}
```

### Decision Framework

| Dataset Size | Optimization |
|-------------|--------------|
| Small (<1k) | Plain `.tracking` |
| Medium (1-10k) | `.removeDuplicates()` + `.debounce()` |
| Large (10k+) | Region tracking |

### Async/Await

```swift
for try await tracks in ValueObservation
    .tracking { db in try Track.fetchAll(db) }
    .values(in: dbQueue) {
    print("Updated: \(tracks.count)")
}
```

---

## GRDB Migrations

### DatabaseMigrator Basics

```swift
import GRDB

var migrator = DatabaseMigrator()

migrator.registerMigration("v1_initial") { db in
    try db.create(table: "tracks") { t in
        t.column("id", .text).primaryKey()
        t.column("title", .text).notNull()
        t.column("artist", .text).notNull()
        t.column("duration", .real).notNull()
    }
}

migrator.registerMigration("v2_add_genre") { db in
    try db.alter(table: "tracks") { t in
        t.add(column: "genre", .text)
    }
}

migrator.registerMigration("v3_add_indexes") { db in
    try db.create(index: "idx_tracks_genre", on: "tracks", columns: ["genre"])
}

try migrator.migrate(dbQueue)
```

### Migration Guarantees

- Each migration runs exactly ONCE
- Migrations run in registration order
- Safe to call `migrate()` multiple times
- Runs in transaction (all or nothing)

No need for `IF NOT EXISTS` - GRDB handles versioning.

### Column Types

| Swift Type | SQLite Type |
|-----------|-------------|
| String | .text |
| Int | .integer |
| Double | .real |
| Data | .blob |
| Date | .datetime |
| Bool | .boolean |

### Creating Tables

```swift
migrator.registerMigration("create_albums") { db in
    try db.create(table: "albums") { t in
        t.column("id", .text).primaryKey()
        t.column("title", .text).notNull()
        t.column("artistId", .text)
            .references("artists", onDelete: .cascade)
        t.column("createdAt", .datetime).defaults(to: Date())
    }
}
```

### Creating Indexes

```swift
// Simple index
try db.create(index: "idx_artist", on: "tracks", columns: ["artist"])

// Compound index
try db.create(index: "idx_genre_duration", on: "tracks", columns: ["genre", "duration"])

// Unique index
try db.create(index: "idx_external_id", on: "tracks", columns: ["externalId"], unique: true)
```

### Data Migrations

```swift
migrator.registerMigration("normalize_artists") { db in
    try db.create(table: "artists") { t in
        t.column("id", .text).primaryKey()
        t.column("name", .text).notNull()
    }

    try db.execute(sql: """
        INSERT INTO artists (id, name)
        SELECT DISTINCT lower(replace(artist, ' ', '_')), artist FROM tracks
        """)

    try db.alter(table: "tracks") { t in
        t.add(column: "artistId", .text).references("artists")
    }

    try db.execute(sql: """
        UPDATE tracks SET artistId = (
            SELECT id FROM artists WHERE artists.name = tracks.artist
        )
        """)
}
```

### Foreign Key Cascade Options

| Option | Behavior |
|--------|----------|
| `.cascade` | Delete children when parent deleted |
| `.setNull` | Set FK to NULL |
| `.restrict` | Prevent deletion if children exist |

### Development Mode

```swift
#if DEBUG
migrator.eraseDatabaseOnSchemaChange = true  // Wipes DB on schema change
#endif
```

### Best Practices

1. Never modify existing migrations - add new ones
2. Test with production data
3. One logical change per migration
4. Use descriptive names: `v5_add_preferences` not `migration_5`

---

## GRDB Performance

### Query Profiling

#### Enable Tracing

```swift
var config = Configuration()
config.trace = { print($0) }
let dbQueue = try DatabaseQueue(path: dbPath, configuration: config)
```

#### EXPLAIN QUERY PLAN

```swift
try dbQueue.read { db in
    let plan = try String.fetchOne(db, sql: """
        EXPLAIN QUERY PLAN SELECT * FROM tracks WHERE artist = ?
        """, arguments: ["Artist"])
    print(plan)
}
```

Key terms:
- **SCAN** - Full table scan (slow)
- **SEARCH** - Uses index (fast)

### Index Strategies

```swift
// Single column
try db.create(index: "idx_artist", on: "tracks", columns: ["artist"])

// Compound (multi-column queries)
try db.create(index: "idx_genre_artist", on: "tracks", columns: ["genre", "artist"])
```

#### When to Index

| Scenario | Example |
|----------|---------|
| WHERE clause | `WHERE artist = ?` |
| JOIN columns | `ON tracks.albumId = albums.id` |
| ORDER BY | `ORDER BY createdAt DESC` |
| Foreign keys | `artistId`, `albumId` |

#### Anti-Patterns

- Don't index booleans or low-cardinality columns
- Don't over-index small tables (<1000 rows)

### Batch Operations

#### Wrong: Many Transactions

```swift
for track in tracks {
    try dbQueue.write { db in try track.insert(db) }  // Slow!
}
```

#### Right: Single Transaction

```swift
try dbQueue.write { db in
    for track in tracks { try track.insert(db) }
}
```

#### Prepared Statements

```swift
try dbQueue.write { db in
    let stmt = try db.makeStatement(sql: "INSERT INTO tracks VALUES (?, ?, ?)")
    for track in tracks {
        try stmt.execute(arguments: [track.id, track.title, track.artist])
    }
}
```

### Avoiding N+1 Queries

#### Wrong

```swift
let tracks = try Track.fetchAll(db)
for track in tracks {
    let album = try Album.fetchOne(db, key: track.albumId)  // N queries!
}
```

#### Right: Use JOIN

```swift
let sql = "SELECT tracks.*, albums.title as albumTitle FROM tracks JOIN albums ON ..."
let results = try TrackWithAlbum.fetchAll(db, sql: sql)
```

### Main Thread Safety

#### Wrong

```swift
let tracks = try dbQueue.read { db in try Track.fetchAll(db) }  // Blocks UI
```

#### Right: Async

```swift
Task {
    let tracks = try await dbQueue.read { db in try Track.fetchAll(db) }
}
```

### Large Datasets

```swift
// Stream instead of loading all
let cursor = try Track.fetchCursor(db)
while let track = try cursor.next() {
    process(track)
}
```

### Profiling Checklist

1. Enable `config.trace`
2. Run `EXPLAIN QUERY PLAN` on slow queries
3. Look for SCAN (add indexes)
4. Wrap batch writes in transactions
5. Check for N+1 patterns
6. Use background threads for heavy reads

---
