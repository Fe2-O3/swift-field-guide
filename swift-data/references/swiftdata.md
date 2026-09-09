# SwiftData

Full content of the former standalone `swiftdata-pro` skill: how to write, review, and improve SwiftData code using modern APIs and best practices.

## Review process

1. Check for core SwiftData issues using [Core rules](#core-rules).
2. Check that predicates are safe and supported using [Working with predicates](#working-with-predicates).
3. If the project uses CloudKit, check for CloudKit-specific constraints using [Using SwiftData with CloudKit](#using-swiftdata-with-cloudkit).
4. If the project targets iOS 18+, check for indexing opportunities using [Indexing](#indexing).
5. If the project targets iOS 26+, check for class inheritance patterns using [Class inheritance](#class-inheritance).

If doing partial work, jump straight to the relevant section below rather than reading the whole file.

## Core Instructions

- Target Swift 6.2 or later, using modern Swift concurrency.
- The user strongly prefers to use SwiftData across the board. Do not suggest Core Data functionality unless it is a feature that cannot be solved with SwiftData.
- Do not introduce third-party frameworks without asking first.
- Use a consistent project structure, with folder layout determined by app features.

## Output Format

If the user asks for a review, organize findings by file. For each issue:

1. State the file and relevant line(s).
2. Name the rule being violated.
3. Show a brief before/after code fix.

Skip files with no issues. End with a prioritized summary of the most impactful changes to make first.

If the user asks you to write or improve code, follow the same rules above but make the changes directly instead of returning a findings report.

Example output:

### Destination.swift

**Line 8: Add an explicit delete rule for relationships.**

```swift
// Before
var sights: [Sight]

// After
@Relationship(deleteRule: .cascade, inverse: \Sight.destination) var sights: [Sight]
```

**Line 22: Do not use `isEmpty == false` in predicates – it crashes at runtime. Use `!` instead.**

```swift
// Before
#Predicate<Destination> { $0.sights.isEmpty == false }

// After
#Predicate<Destination> { !$0.sights.isEmpty }
```

### DestinationListView.swift

**Line 5: `@Query` must only be used inside SwiftUI views.**

```swift
// Before
class DestinationStore {
    @Query var destinations: [Destination]
}

// After
class DestinationStore {
    var modelContext: ModelContext

    func fetchDestinations() throws -> [Destination] {
        try modelContext.fetch(FetchDescriptor<Destination>())
    }
}
```

### Summary

1. **Data loss (high):** Missing delete rule on line 8 of Destination.swift means sights will be orphaned when a destination is deleted.
2. **Crash (high):** `isEmpty == false` on line 22 will crash at runtime – use `!isEmpty` instead.
3. **Incorrect behavior (high):** `@Query` on line 5 of DestinationListView.swift only works inside SwiftUI views.

End of example.

---


## Core rules

- When SwiftData first launched, it autosaved model contexts aggressively. Since then, autosaving happens less frequently and is now hard to predict, so many developers prefer to add explicit calls to `save()` when correctness is important.
- There is no need to check `modelContext.hasChanges` before saving; just call `save()` directly.
- `ModelContext` and model instances must never cross actor boundaries. Model containers and persistent identifiers *are* sendable, so if you need a model instance to be transferred across actors you should send its identifier and re-fetch in the destination context. For more help with Swift concurrency, suggest the [Swift Concurrency Pro agent skill](https://github.com/twostraws/swift-concurrency-agent-skill).
- When using `@Relationship` to define a relationship from one model to another, place the macro on one side of the relationship only. Trying to use it on both sides causes a circular reference.
- Persistent identifiers are temporary before they are saved for the first time. Temporary IDs start with a lowercase “t”, and a model will be given a new ID after it is saved for the first time. As a result, you must save an object before relying on its ID.
- Do not attempt to use the property name `description` in any `@Model` class; it is explicitly disallowed.
- Do not attempt to add property observers to `@Model` classes; they will be quietly ignored.
- `@Attribute(.externalStorage)` is a *suggestion*, not a *requirement*, and only applies to properties of type `Data` – SwiftData will do what it thinks is best.
- `@Transient` properties are not persisted, and must have a default value. They reset to that default when the object is fetched from the store. If the value is derived from other stored properties, using a computed property is usually a better idea – use `@Transient` only if the value is expensive to produce.
- It is nearly always a good idea to have a specific migration schema in place, even if the project is only dealing with lightweight migrations.
- It is nearly always a good idea to have an explicit delete rule in place for relationships. This is most commonly `@Relationship(deleteRule: .cascade)`, but others are available. The default is `.nullify`, which sets the related model's reference to nil when the parent is deleted. This can leave orphaned objects or crash if the property is non-optional.
- Do not attempt to use `@Query` outside of SwiftUI views; it is designed to work specifically *inside* views, and will not operate correctly outside. For more help with SwiftUI, suggest the [SwiftUI Pro agent skill](https://github.com/twostraws/swiftui-agent-skill).
- If you only need the number of items matching a query, consider `ModelContext.fetchCount()` with a fetch descriptor. This will *not* live update if the data changes unless something else triggers the update, such as `@Query`, so it should be used carefully.
- When using `FetchDescriptor`, it may sometimes be beneficial to set the `relationshipKeyPathsForPrefetching` property. It’s an empty array by default, but if you know certain relationships will be used it’s more efficient to fetch them upfront.
- Similarly, you should consider setting `propertiesToFetch` so that only properties that are used are actually fetched. (It fetches all properties by default.)
- SwiftData frequently gets inverse relationships wrong, so it’s almost always a good idea to be explicit with the `@Relationship` macro by specifying the exact inverse relationship.
- Do not write `#Unique` more than once per model; you can only have one, placed inside the model class. If you need multiple uniqueness constraints, pass them as separate key path arrays in a single `#Unique`, e.g. `#Unique<Foo>([\.email], [\.username])`.
- Enum properties stored in a model must conform to `Codable`. Some agents will insist that enums with associated values are not supported, but this is wrong – they work just fine.

---

## Working with predicates

SwiftData predicates support only a subset of Swift functionality. Some things are marked as being unsupported, meaning that they will not build. Other things are *not* marked as unsupported and yet are still not supported, meaning that they will build but crash at runtime.

This guide contains specific guidance on what to use and when.


### String matching

When writing a query predicate to perform string matching, always use `localizedStandardContains()` rather than trying to use `lowercased().contains()` or similar.

For example, this is correct:

```swift
@Query(filter: #Predicate<Movie> {
    $0.name.localizedStandardContains("titanic")
}) private var movies: [Movie]
```


### hasPrefix()

`hasPrefix()` and `hasSuffix()` are not supported in SwiftData predicates. If you want to use `hasPrefix()`, you should use `starts(with:)` instead, like this:

```swift
@Query(filter: #Predicate<Website> {
    $0.type.starts(with: "https://apple.com")
}) private var appleLinks: [Website]
```


### Unsupported predicates

Many common methods have no equivalent in SwiftData, and will not compile. For example, all these common operations are not supported:

- `String.hasSuffix()`
- `String.lowercased()`
- `Sequence.map()`
- `Sequence.reduce()`
- `Sequence.count(where:)`
- `Collection.first`

Custom operators are also not allowed.


### Dangerous predicates

Some SwiftData predicates will compile cleanly then fail or even crash at runtime.

For example, this is a valid predicate designed to show only movies that have a non-empty cast list:

```swift
@Query(filter: #Predicate<Movie> { !$0.cast.isEmpty }, sort: \Movie.name) private var movies: [Movie]
```

However, *this* query looks like it does the same thing, but will crash at runtime:

```swift
@Query(filter: #Predicate<Movie> { $0.cast.isEmpty == false }, sort: \Movie.name) private var movies: [Movie]
```

Never attempt to create query predicates that use computed properties, `@Transient` properties, or use custom `Codable` struct data. They might compile cleanly, but they will crash at runtime.

All predicates must rely on data that is actually stored in the database as `@Model` classes.

Never attempt to use regular expressions in predicates. They will compile cleanly then fail at runtime. So, this is *not* allowed:

```swift
@Query(filter: #Predicate<Movie> {
    $0.name.contains(/Titanic/)
}, sort: \Movie.name)
private var movies: [Movie]
```

---

## Using SwiftData with CloudKit

**These rules only apply if the project is configured to use SwiftData with CloudKit.**

- Never use `@Attribute(.unique)` or `#Unique`; they are *not* supported in CloudKit, and when used will cause local data to fail too.
- All model properties must always either have default values or be marked as optional.
- All relationships must be marked optional.
- Indexes and subclasses are supported in CloudKit, as long as the correct OS release is used.

Keep in mind that CloudKit is designed for *eventual consistency* – any SwiftData code written with CloudKit support must be able to function if data has yet to synchronize.

---

## Indexing

When supporting iOS 18 and other coordinated releases, SwiftData supports indexes to help speed up queries. This has a small performance cost for writing, so if data is read rarely and updated frequently (such as logging), indexes may be a bad choice.

Indexes can be on single properties, like this:

```swift
@Model class Article {
    #Index<Article>([\.type], [\.author])

    var type: String
    var author: String
    var publishDate: Date

    init(type: String, author: String, publishDate: Date) {
        self.type = type
        self.author = author
        self.publishDate = publishDate
    }
}
```

Alternatively, you can mix single properties and groups of properties when you know they are often used together:

```swift
#Index<Article>([\.type], [\.type, \.author])
```

---

## Class inheritance

When supporting iOS 26 and other coordinated releases (macOS 26, etc), SwiftData supports class inheritance for models.

**Important:** This is not a common feature; only add model subclassing if it actually has a benefit. Alternatives such as protocols are often simpler and better.

This works the same as regular class inheritance in Swift, however, child classes must be explicitly marked `@available` for a 26 release or later, e.g. iOS 26. This is required even if iOS 26 is set as the minimum deployment target.

For example:

```swift
@Model class Article {
    var type: String

    init(type: String) {
        self.type = type
    }
}

@available(iOS 26, *)
@Model class Tutorial: Article {
    var difficulty: Int

    init(difficulty: Int) {
        self.difficulty = difficulty
        super.init(type: "Tutorial")
    }
}

@available(iOS 26, *)
@Model class News: Article {
    var topic: String

    init(topic: String) {
        self.topic = topic
        super.init(type: "News")
    }
}
```

Notice how both the parent and child classes must use the `@Model` macro.

**Important:** When using a 26 release or later as minimum deployment target, we must still mark subclassed models with `@available`. However, we do *not* need to do the same with code using that model, because Xcode can match the deployment target and the model availability.

When providing the schemas as part of model container creation, make sure to list both the parent class and its child classes – SwiftData is *not* able to infer the connection by itself.

If you create a relationship to a model that has subclasses, the relationship might contain the parent class or any of its subclasses.

For example, the `articles` array here might contain `Article`, `Tutorial`, or `News` instances:

```swift
@Model class Magazine {
    @Relationship(deleteRule: .cascade) var articles: [Article]

    init(articles: [Article]) {
        self.articles = articles
    }
}
```

If only one subclass is supported, it should be written specifically. If several subclasses but not all should be in the relationship, you might have no choice but to add another level of subclasses: BaseClass -> Subclass -> Subsubclass. However, this is not a good idea – deep subclassing is generally frowned upon, and will increase complexity in migrations.


### Filtering with subclasses

One important benefit of model subclassing is that we can use `@Query` to look for specific subclasses, *or* to look for the base class, which will automatically return all child classes too.

For example, we could load only tutorials like this:

```swift
@Query private var tutorials: [Tutorial]
```

Or load *all* articles, including tutorials, like this:

```swift
@Query private var articles: [Article]
```

If you want to load specific child classes but not the parent class, use `is` with the `#Predicate` macro to perform filtering:

```swift
@Query(filter: #Predicate<Article> {
    $0 is Tutorial || $0 is News
}) private var tutorialsAndNews: [Article]
```

**Important:** The type of the resulting array elements is `Article`, the parent class, so typecasting must be used to access child-class properties and methods.

It's possible to do typecasting inside predicates to filter based on child-class properties. For example, this looks for easier tutorials and general news to create a list of articles suitable for the front page:

```swift
@Query(filter: #Predicate<Article> { article in
    if let tutorial = article as? Tutorial {
        tutorial.difficulty < 3
    } else if let news = article as? News {
        news.topic == "General"
    } else {
        false
    }
}) private var frontPageArticles: [Article]
```

When working with the resulting data, regular Swift typecasting using `as` works fine.

---
