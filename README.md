# Swift Field Guide

**17 battle-tested agent skills for Swift, SwiftUI, and iOS development.**

Built for Claude Code, Codex, OpenCode, and any agent that reads `SKILL.md` files.

These are not prompt collections. Each skill is a reference library — decision trees, code patterns,
migration guides, and checklists an agent loads *before* it starts guessing from memory.

## Why fewer, bigger skills

Most skill collections suffer the same disease: ten overlapping skills, stale APIs, no clear
"which one loads when." This repo went the other way. Roughly forty overlapping skills from five
sources were merged, de-duplicated, and brought current into seventeen. Examples:

| Before | After |
|---|---|
| 5 overlapping SwiftUI skills | `swiftui` — one skill, ~30 reference files |
| 4 persistence skills (Core Data, SwiftData, SQLiteData, GRDB) | `swift-data` — including *choosing between them* |
| Separate concurrency, migration, and review skills | `swift-concurrency` — 18 references, one router |
| Ad-hoc build-speed advice | `xcode-build` + `spm-build-analysis` — benchmark-driven |

Where merged sources genuinely disagreed, the references say so explicitly instead of presenting a
false consensus.

## The catalog

### Language

| Skill | What it does |
|---|---|
| **write-swift** | How to write modern Swift well: value types, Swift 6 data-race safety, `@concurrent`, actors, protocols and generics (`some` vs `any`), API design, performance and ARC, macros. |
| **swift-concurrency** | Tasks, actors, `@MainActor`, `Sendable`, data races, cancellation, `TaskGroup` patterns, Swift 6 migration, and concurrency code review. |
| **swift-style** | Swift code conventions: formatting, naming, organization, idiomatic patterns. |

### UI

| Skill | What it does |
|---|---|
| **swiftui** | Writing, reviewing, and refactoring SwiftUI: state management, gesture composition, adaptive layout, navigation, architecture choice (MVVM vs TCA vs vanilla), Liquid Glass, accessibility — plus Instruments `.trace` capture/analysis for hangs and hitches. |
| **ios-hig** | Human Interface Guidelines compliance: accessibility, Dynamic Type, dark mode, 44pt touch targets, animation and haptics, permissions. |
| **haptics** | `UIFeedbackGenerator` and Core Haptics patterns for confirmations, errors, and custom tactile experiences. |

### Platform

| Skill | What it does |
|---|---|
| **ios-26-platform** | iOS 26 features: Liquid Glass, new SwiftUI APIs, `WebView`, `Chart3D` — and backward compatibility with iOS 17/18. |
| **localization** | String Catalogs, pluralization, right-to-left layout, modern i18n workflows. |

### Data

| Skill | What it does |
|---|---|
| **swift-data** | Core Data, SwiftData, SQLiteData, and GRDB — stack setup, threading, migrations, CloudKit sync, and *which framework to pick*. |

### Testing

| Skill | What it does |
|---|---|
| **swift-testing** | `@Test`, `#expect`, `#require`, traits, parameterized tests, parallel execution, and XCTest → Swift Testing migration. |

### Diagnostics & performance

| Skill | What it does |
|---|---|
| **swift-diagnostics** | Systematic decision trees for NavigationStack bugs, build failures, and memory leaks — diagnosis in minutes, not hours. |
| **xcode-build** | The single entry point for build-speed work: benchmark clean/incremental builds, diagnose slow type-checking, audit settings, prove the win with before/after numbers. |
| **spm-build-analysis** | SPM dependency graphs, package plugins, module variants, CI overhead, and modularization strategy. |
| **swift-networking** | Network.framework: `NWConnection`, UDP/TCP patterns, structured-concurrency networking, migrating off raw sockets. |

### Ecosystem

| Skill | What it does |
|---|---|
| **composable-architecture** | TCA: `@Reducer`, `Store`, `Effect`, `TestStore`, reducer composition, effect handling. |
| **storekit** | StoreKit 2: subscriptions, consumables, `.storekit` configuration testing, `StoreManager` architecture, transaction verification. |
| **generating-swift-package-docs** | On-demand API documentation for unfamiliar package dependencies. |

## Install

### Claude Code

```bash
git clone https://github.com/Fe2-O3/swift-field-guide.git
cd swift-field-guide
mkdir -p ~/.claude/skills
for d in */; do cp -R "$d" ~/.claude/skills/; done
```

Every top-level directory in this repo is a skill, so the loop copies exactly the seventeen.

Or install globally to `~/.claude/skills/`, or per-project to `<project>/.claude/skills/`.

### Codex / OpenCode / other agents

Any harness that reads the `SKILL.md` agent-skills format can use these:
point it at a cloned copy of this repo, or symlink the skills you want.

```bash
ln -s /path/to/swift-field-guide/swiftui ~/.codex/skills/swiftui
```

### Recommended set

Start with these four — they cover most Swift work an agent does:

**write-swift · swiftui · swift-concurrency · swift-diagnostics**

## FAQ

**Why is the animation suite not in this repo?**
Animation craft skills (and others in active development) are kept separate. This repo is the
stable, Swift-and-iOS core.

**Does this replace [AvdLee's per-topic skills](https://github.com/AvdLee)?**
It overlaps. Antoine's skills are excellent and this repo credits his work where it's used
(see [ATTRIBUTION.md](ATTRIBUTION.md)). The difference is consolidation: fewer skills, one
reference router each, and disagreements between sources documented rather than averaged.

**How current are the APIs?**
The `swiftui` skill ships a deprecation-tracking reference (`latest-apis.md`) plus a refresh
workflow that scans Apple's docs via the Sosumi MCP after each Xcode release.

## License

[MIT](LICENSE). See [ATTRIBUTION.md](ATTRIBUTION.md) for third-party credits.
