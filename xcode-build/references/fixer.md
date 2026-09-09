# Fixer Reference

Full content of the former standalone `xcode-build-fixer` skill, merged in as a reference for [xcode-build](../SKILL.md). Use this to implement approved build optimization changes and verify them with a benchmark.

## Core Rules

- Only apply changes that have explicit developer approval.
- Apply one logical fix at a time so changes are reviewable and reversible.
- Re-benchmark after applying changes to verify improvement.
- Report exactly what changed, which files were touched, and the measured delta.
- If a change produces no improvement or causes a regression, flag it immediately.

## Inputs

The fixer expects one of:

- An approved optimization plan at `.build-benchmark/optimization-plan.md` with checked approval boxes.
- An explicit developer instruction describing the fix to apply (e.g., "set `DEBUG_INFORMATION_FORMAT` to `dwarf` for Debug").

When working from an optimization plan, read the approval checklist and implement only the checked items.

## Fix Categories

### Build Settings

Change `project.pbxproj` values to match the recommendations in [Build Settings Best Practices](#build-settings-best-practices-full-reference) below.

Typical fixes:

- Set `DEBUG_INFORMATION_FORMAT = dwarf` for Debug
- Set `SWIFT_COMPILATION_MODE = singlefile` for Debug
- Enable `COMPILATION_CACHE_ENABLE_CACHING = YES`
- Enable `EAGER_LINKING = YES` for Debug
- Align cross-target settings to eliminate module variants

When editing `project.pbxproj`, locate the correct `buildSettings` block by matching the target name and configuration name. Verify the change with `xcodebuild -showBuildSettings` after applying.

### Script Phases

Fix run script phases that waste time during incremental or debug builds.

Typical fixes:

- Add input and output file declarations so Xcode can skip unchanged scripts.
- Add configuration guards: `[[ "$CONFIGURATION" != "Release" ]] && exit 0` for release-only scripts.
- Move input/output lists into `.xcfilelist` files when the list is long.
- Enable `Based on dependency analysis` when inputs and outputs are declared.

### Source-Level Compilation Fixes

Apply code changes that reduce type-checker and compiler overhead. See [Fix Patterns](#fix-patterns-full-reference) below for before/after patterns.

Typical fixes:

- Add explicit type annotations to complex expressions.
- Break long chained or nested expressions into intermediate typed variables.
- Mark classes `final` when they are not subclassed.
- Tighten access control (`private`/`fileprivate`) for internal-only symbols.
- Extract monolithic SwiftUI `body` properties into smaller composed subviews.
- Replace deeply nested result-builder code with separate typed helpers.
- Add explicit return types to closures passed to generic functions.

### SPM Restructuring

Restructure Swift packages to improve build parallelism and reduce rebuild scope.

Typical fixes:

- Move shared types to a lower-layer module to eliminate circular or upward dependencies.
- Split oversized modules (200+ files) by feature area.
- Extract protocol definitions into lightweight interface modules.
- Remove unnecessary `@_exported import` usage.
- Align build options across targets that import the same packages to prevent module variant duplication.
- Pin branch-tracked dependencies to tagged versions or commit hashes for deterministic resolution.

Before applying version pin changes:

- Run `git ls-remote --tags <url>` to confirm tags exist. If the upstream has no tags, pin to a specific revision hash instead.
- Verify the pinned version resolves successfully with `xcodebuild -resolvePackageDependencies` before proceeding.

## Execution Workflow

1. Read the approved optimization plan or developer instruction.
2. For each approved item, identify the exact files and locations to change.
3. Apply the change.
4. Verify the change compiles: run a quick `xcodebuild build` to confirm no errors were introduced.
5. After all approved changes are applied, re-benchmark using the same inputs from the original baseline:
   ```bash
   python3 scripts/benchmark_builds.py \
     --project App.xcodeproj \
     --scheme MyApp \
     --configuration Debug \
     --destination "platform=iOS Simulator,name=iPhone 16" \
     --output-dir .build-benchmark
   ```
6. Compare post-change medians to the baseline and report deltas.

## Evaluating Regressions

Not every slower number is a true regression. The fixer must evaluate the full picture before recommending a revert.

### Compilation caching trade-off

A change like `COMPILATION_CACHE_ENABLE_CACHING = YES` may make a standard clean build slightly slower (cache population overhead) while making cached clean builds significantly faster. Since cached clean builds reflect the realistic developer workflow (branch switching, pulling changes, Clean Build Folder with a warm cache), a slower standard clean build paired with a faster cached clean build is a net improvement, not a regression. The same logic applies to any change where the first-time cost is higher but subsequent builds benefit.

### Compare all build types

Evaluate standard clean, cached clean (when available), and incremental builds together. A change is only a clear regression if it makes the metrics that matter to the developer's daily workflow worse. If only the cold-start clean build is slower but the developer rarely does truly cold builds, that may be acceptable.

### Per-change vs. cumulative

A single change that regresses on its own may still be worth keeping if it enables a larger cumulative improvement (e.g., enabling compilation caching makes one build type slower but unlocks cache benefits for everything else).

### Best-practice settings are not revert candidates

Some build settings are Apple's recommended modern defaults. These should be applied and kept regardless of whether the immediate benchmark shows a measurable improvement. They are not speculative optimizations -- they align the project with Apple's toolchain direction and will compound in value over future Xcode releases. The fixer must never recommend reverting these settings based on a single benchmark result.

Best-practice settings that should always be kept once applied:

- `COMPILATION_CACHE_ENABLE_CACHING = YES` -- Apple is actively investing in this; the cache improves with each Xcode release and compounds across real workflows
- `EAGER_LINKING = YES` (Debug) -- allows the linker to overlap with compilation
- `SWIFT_USE_INTEGRATED_DRIVER = YES` -- eliminates inter-process scheduling overhead
- `DEBUG_INFORMATION_FORMAT = dwarf` (Debug) -- avoids unnecessary dSYM generation
- `SWIFT_COMPILATION_MODE = singlefile` (Debug) -- incremental recompilation
- `ONLY_ACTIVE_ARCH = YES` (Debug) -- no reason to build all architectures locally

When reporting on these settings, use language like: "Applied recommended build setting. No immediate benchmark improvement measured, but this aligns with Apple's recommended configuration and positions the project for future Xcode improvements."

### When to recommend revert (speculative changes only)

For changes that are not best-practice settings (e.g., source refactors, linkage experiments, script phase modifications, dependency restructuring):

- If the cumulative pass shows wall-clock regression across all measured build types (standard clean, cached clean, and incremental are all slower), recommend reverting all speculative changes unless the developer explicitly asks to keep specific items for non-performance reasons.
- For each individual speculative change: if it shows no median improvement and no cached/incremental benefit either, flag it with `Recommend revert` and the measured delta.
- Distinguish between "outlier reduction only" (improved worst-case but not median) and "median improvement" (improved typical developer wait).
- When a change trades off one build type for another (e.g., slower standard clean but faster cached clean), present both numbers clearly and let the developer decide. Frame it as: "Standard clean builds are X.Xs slower, but cached clean builds (the realistic daily workflow) are Y.Ys faster."

## Reporting

Lead with the wall-clock result in plain language:

> "Your clean build now takes X.Xs (was Y.Ys) -- Z.Zs faster."
> "Your incremental build now takes X.Xs (was Y.Ys) -- Z.Zs faster."

Then include:

- Post-change clean build wall-clock median
- Post-change incremental build wall-clock median
- Absolute and percentage wall-clock deltas for both
- Confidence notes if benchmark noise is high
- List of files modified per fix
- Any deviations from the original recommendation

If cumulative task metrics improved but wall-clock did not, say plainly: "Compiler workload decreased but build wait time did not improve. This is expected when Xcode runs these tasks in parallel with other equally long work."

If a fix produced no measurable wall-time improvement, note `No measurable wall-time improvement` and suggest whether to keep (e.g. for code quality) or revert.

For changes valuable for non-benchmark reasons (deterministic package resolution, branch-switch caching), label them: "No wait-time improvement expected from this change. The benefit is [deterministic builds / faster branch switching / reduced CI cost]."

Note: `COMPILATION_CACHE_ENABLE_CACHING` has been measured at 5-14% faster clean builds across tested projects (87 to 1,991 Swift files). The benefit compounds in real developer workflows where the cache persists between builds -- branch switching, pulling changes, and CI with persistent DerivedData. The benchmark script auto-detects this setting and runs a cached clean phase for validation.

## Execution Report

After the optimization pass is complete, produce a structured execution report. This gives the developer a clear summary of what was attempted, what worked, and what the final state is.

Structure:

```markdown
## Execution Report

### Baseline
- Clean build median: X.Xs
- Cached clean build median: X.Xs (if applicable)
- Incremental build median: X.Xs

### Changes Applied

| # | Change | Actionability | Measured Result | Status |
|---|--------|---------------|-----------------|--------|
| 1 | Description | repo-local | Clean: X.Xs→Y.Ys, Incr: X.Xs→Y.Ys | Kept / Reverted / Blocked |
| 2 | ... | ... | ... | ... |

### Final Cumulative Result
- Clean build median: X.Xs (was Y.Ys) -- Z.Zs faster/slower
- Cached clean build median: X.Xs (was Y.Ys) -- Z.Zs faster/slower
- Incremental build median: X.Xs (was Y.Ys) -- Z.Zs faster/slower
- **Net result:** Faster / Slower / Unchanged

### Blocked or Non-Actionable Findings
- Finding: reason it could not be addressed from the repo
```

Status values:

- `Kept` -- Change improved or maintained build times and was kept.
- `Kept (best practice)` -- Change is a recommended build setting; kept regardless of immediate benchmark result.
- `Reverted` -- Change regressed build times and was reverted.
- `Blocked` -- Change could not be applied due to project structure, Xcode behavior, or external constraints.
- `No improvement` -- Change compiled but showed no measurable wall-time benefit. Include whether it was kept (for non-performance reasons) or reverted.

## Escalation

If during implementation you discover issues outside this reference's scope:

- Project-level analysis gaps: see [project-analysis.md](project-analysis.md)
- Compilation hotspot analysis: see [compilation-analysis.md](compilation-analysis.md)
- Package graph issues: see [`spm-build-analysis`](../../spm-build-analysis/SKILL.md) (a separate, standalone skill)

## Fix Patterns (Full Reference)

Concrete before/after examples for each fix category. Reference this when applying approved changes to ensure consistency.

### Build Settings Fixes

#### Debug Information Format (Debug)

Before (`project.pbxproj`):
```
DEBUG_INFORMATION_FORMAT = "dwarf-with-dsym";
```

After:
```
DEBUG_INFORMATION_FORMAT = dwarf;
```

#### Compilation Mode (Debug)

Before:
```
SWIFT_COMPILATION_MODE = wholemodule;
```

After:
```
SWIFT_COMPILATION_MODE = singlefile;
```

#### Enable Compilation Caching

Before (setting absent or):
```
COMPILATION_CACHE_ENABLE_CACHING = NO;
```

After:
```
COMPILATION_CACHE_ENABLE_CACHING = YES;
```

#### Enable Eager Linking (Debug)

Before (setting absent):

After:
```
EAGER_LINKING = YES;
```

### Script Phase Fixes

#### Add Configuration Guard

Before:
```bash
# Upload dSYMs to crash reporter
./scripts/upload-dsyms.sh
```

After:
```bash
# Upload dSYMs to crash reporter
[[ "$CONFIGURATION" != "Release" ]] && exit 0
./scripts/upload-dsyms.sh
```

#### Add Input/Output Declarations

When a script has no declared inputs or outputs, Xcode runs it on every build. Declare them in the build phase or use `.xcfilelist` files for long lists.

Before (in Xcode build phase):
```
Input Files: (none)
Output Files: (none)
```

After:
```
Input Files:
  $(SRCROOT)/scripts/generate-constants.sh
  $(SRCROOT)/Config/constants.json

Output Files:
  $(DERIVED_FILE_DIR)/GeneratedConstants.swift
```

### Source-Level Fixes

#### Add Explicit Type Annotations

Before:
```swift
let result = items.map { $0.value }.filter { $0 > threshold }.reduce(0, +)
```

After:
```swift
let mapped: [Double] = items.map { $0.value }
let filtered: [Double] = mapped.filter { $0 > threshold }
let result: Double = filtered.reduce(0, +)
```

#### Break Complex Expressions

Before:
```swift
let config = try JSONDecoder().decode(
    AppConfig.self,
    from: Data(contentsOf: Bundle.main.url(forResource: "config", withExtension: "json")!)
)
```

After:
```swift
let configURL: URL = Bundle.main.url(forResource: "config", withExtension: "json")!
let configData: Data = try Data(contentsOf: configURL)
let config: AppConfig = try JSONDecoder().decode(AppConfig.self, from: configData)
```

#### Mark Classes Final

Before:
```swift
class NetworkService {
    func fetchData() async throws -> Data { ... }
}
```

After:
```swift
final class NetworkService {
    func fetchData() async throws -> Data { ... }
}

```

Only apply when the class is not subclassed anywhere in the project. Search for `: NetworkService` and `class ... : NetworkService` before marking `final`.

#### Tighten Access Control

Before:
```swift
class ViewModel {
    var internalState: State = .idle
    func processQueue() { ... }
}
```

After:
```swift
class ViewModel {
    private var internalState: State = .idle
    private func processQueue() { ... }
}
```

Apply `private` when the symbol is only used within the same declaration. Apply `fileprivate` when used within the same file but outside the declaration.

#### Extract SwiftUI Subviews

Before:
```swift
struct ContentView: View {
    var body: some View {
        VStack {
            HStack {
                Image(systemName: "person")
                Text(user.name)
                Spacer()
                Button("Edit") { showEdit = true }
            }
            List(items) { item in
                HStack {
                    Text(item.title)
                    Spacer()
                    Text(item.subtitle)
                        .foregroundStyle(.secondary)
                }
            }
        }
    }
}
```

After:
```swift
struct ContentView: View {
    var body: some View {
        VStack {
            UserHeaderView(user: user, showEdit: $showEdit)
            ItemListView(items: items)
        }
    }
}

struct UserHeaderView: View {
    let user: User
    @Binding var showEdit: Bool

    var body: some View {
        HStack {
            Image(systemName: "person")
            Text(user.name)
            Spacer()
            Button("Edit") { showEdit = true }
        }
    }
}

struct ItemListView: View {
    let items: [Item]

    var body: some View {
        List(items) { item in
            ItemRowView(item: item)
        }
    }
}

struct ItemRowView: View {
    let item: Item

    var body: some View {
        HStack {
            Text(item.title)
            Spacer()
            Text(item.subtitle)
                .foregroundStyle(.secondary)
        }
    }
}
```

#### Add Explicit Closure Return Types

Before:
```swift
let handler = { value in
    guard let result = try? process(value) else { return nil }
    return result.transformed()
}
```

After:
```swift
let handler: (InputType) -> OutputType? = { (value: InputType) -> OutputType? in
    guard let result = try? process(value) else { return nil }
    return result.transformed()
}
```

### SPM Restructuring Fixes

#### Extract Shared Types to Lower-Layer Module

Before (`Package.swift`):
```swift
.target(name: "FeatureA", dependencies: ["FeatureB"]),
.target(name: "FeatureB", dependencies: ["FeatureA"]),
```

After:
```swift
.target(name: "SharedContracts", dependencies: []),
.target(name: "FeatureA", dependencies: ["SharedContracts"]),
.target(name: "FeatureB", dependencies: ["SharedContracts"]),
```

Move the shared protocols and types into `SharedContracts` so both features depend downward instead of on each other.

#### Extract Interface Module

Before:
```swift
.target(name: "Networking", dependencies: ["Models"]),
.target(name: "FeatureA", dependencies: ["Networking"]),
.target(name: "FeatureB", dependencies: ["Networking"]),
```

After:
```swift
.target(name: "NetworkingInterface", dependencies: []),
.target(name: "Networking", dependencies: ["NetworkingInterface", "Models"]),
.target(name: "FeatureA", dependencies: ["NetworkingInterface"]),
.target(name: "FeatureB", dependencies: ["NetworkingInterface"]),
```

Feature modules compile against the lightweight interface without waiting for the full implementation to build.

## Build Settings Best Practices (Full Reference)

This reference lists Xcode build settings that affect build performance. Use it to audit a project and produce a pass/fail checklist.

The scope is strictly **build performance**. Do not flag language-migration settings like `SWIFT_STRICT_CONCURRENCY` or `SWIFT_UPCOMING_FEATURE_*` -- those are developer adoption choices unrelated to build speed.

### How To Read This Reference

Each setting includes:

- **Setting name** and the Xcode build-settings key
- **Recommended value** for Debug and Release
- **Why it matters** for build time
- **Risk** of changing it

Use checkmark and cross indicators when reporting:

- `[x]` -- setting matches the recommended value
- `[ ]` -- setting does not match; include the actual value and the expected value

### Debug Configuration

These settings optimize for fast iteration during development.

#### Compilation Mode

- **Key:** `SWIFT_COMPILATION_MODE`
- **Recommended:** `singlefile` (Xcode UI: "Incremental"; or unset -- Xcode defaults to singlefile for Debug)
- **Why:** Single-file mode recompiles only changed files. `wholemodule` recompiles the entire target on every change.
- **Risk:** Low

#### Swift Optimization Level

- **Key:** `SWIFT_OPTIMIZATION_LEVEL`
- **Recommended:** `-Onone`
- **Why:** Optimization passes add significant compile time. Debug builds do not benefit from runtime speed improvements.
- **Risk:** Low

#### GCC Optimization Level

- **Key:** `GCC_OPTIMIZATION_LEVEL`
- **Recommended:** `0`
- **Why:** Same rationale as Swift optimization level, but for C/C++/Objective-C sources.
- **Risk:** Low

#### Build Active Architecture Only

- **Key:** `ONLY_ACTIVE_ARCH` (`BUILD_ACTIVE_ARCHITECTURE_ONLY`)
- **Recommended:** `YES`
- **Why:** Building all architectures doubles or triples compile and link time for no debug benefit.
- **Risk:** Low

#### Debug Information Format

- **Key:** `DEBUG_INFORMATION_FORMAT`
- **Recommended:** `dwarf`
- **Why:** `dwarf-with-dsym` generates a separate dSYM bundle which adds overhead. Plain `dwarf` embeds debug info directly in the binary, which is sufficient for local debugging.
- **Risk:** Low

#### Enable Testability

- **Key:** `ENABLE_TESTABILITY`
- **Recommended:** `YES`
- **Why:** Required for `@testable import`. Adds minor overhead by exporting internal symbols, but this is expected during development.
- **Risk:** Low

#### Active Compilation Conditions

- **Key:** `SWIFT_ACTIVE_COMPILATION_CONDITIONS`
- **Recommended:** Should include `DEBUG`
- **Why:** Guards conditional compilation blocks (e.g., `#if DEBUG`) and ensures debug-only code paths are included.
- **Risk:** Low

#### Eager Linking

- **Key:** `EAGER_LINKING`
- **Recommended:** `YES`
- **Why:** Allows the linker to start work before all compilation tasks finish, reducing wall-clock build time. Particularly effective for Debug builds where link time is a meaningful fraction of total build time.
- **Risk:** Low

### Release Configuration

These settings optimize for production builds.

#### Compilation Mode

- **Key:** `SWIFT_COMPILATION_MODE`
- **Recommended:** `wholemodule`
- **Why:** Whole-module optimization produces faster runtime code. Build time is secondary for release.
- **Risk:** Low

#### Swift Optimization Level

- **Key:** `SWIFT_OPTIMIZATION_LEVEL`
- **Recommended:** `-O` or `-Osize`
- **Why:** Produces optimized binaries. `-Osize` trades some speed for smaller binary size.
- **Risk:** Low

#### GCC Optimization Level

- **Key:** `GCC_OPTIMIZATION_LEVEL`
- **Recommended:** `s`
- **Why:** Optimizes C/C++/Objective-C for size, matching the typical release expectation.
- **Risk:** Low

#### Build Active Architecture Only

- **Key:** `ONLY_ACTIVE_ARCH`
- **Recommended:** `NO`
- **Why:** Release builds must include all supported architectures for distribution.
- **Risk:** Low

#### Debug Information Format

- **Key:** `DEBUG_INFORMATION_FORMAT`
- **Recommended:** `dwarf-with-dsym`
- **Why:** dSYM bundles are required for crash symbolication in production.
- **Risk:** Low

#### Enable Testability

- **Key:** `ENABLE_TESTABILITY`
- **Recommended:** `NO`
- **Why:** Removes internal-symbol export overhead from release builds. Testing should use Debug configuration.
- **Risk:** Low

### General (All Configurations)

#### Compilation Caching

- **Key:** `COMPILATION_CACHE_ENABLE_CACHING`
- **Recommended:** `YES`
- **Why:** Caches compilation results for Swift and C-family sources so repeated compilations of the same inputs are served from cache. The biggest wins come from branch switching and clean builds where source files are recompiled unchanged. This is an opt-in feature. The umbrella setting controls both `SWIFT_ENABLE_COMPILE_CACHE` and `CLANG_ENABLE_COMPILE_CACHE` under the hood; those can be toggled independently if needed.
- **Measurement:** Measured 5-14% faster clean builds across tested projects (87 to 1,991 Swift files). The benefit compounds in real developer workflows where the cache persists between builds -- branch switching, pulling changes, and CI with persistent DerivedData -- though the exact savings depend on how many files change between builds.
- **Risk:** Low -- can also be enabled via per-user project settings so it does not need to be committed to the shared project file.

#### Integrated Swift Driver

- **Key:** `SWIFT_USE_INTEGRATED_DRIVER`
- **Recommended:** `YES`
- **Why:** Uses the integrated Swift driver which runs inside the build system process, eliminating inter-process overhead for compilation scheduling. Enabled by default in modern Xcode but worth verifying in migrated projects.
- **Risk:** Low

#### Clang Module Compilation

- **Key:** `CLANG_ENABLE_MODULES`
- **Recommended:** `YES`
- **Why:** Enables Clang module compilation for C/Objective-C sources, caching module maps on disk instead of reprocessing headers on every import. Eliminates redundant header parsing across translation units.
- **Risk:** Low

#### Explicit Module Builds

- **Key:** `SWIFT_ENABLE_EXPLICIT_MODULES` (C/ObjC enabled by default in Xcode 16+; for Swift use `_EXPERIMENTAL_SWIFT_EXPLICIT_MODULES`)
- **Recommended:** Evaluate per-project
- **Why:** Makes module compilation visible to the build system as discrete tasks, improving parallelism and scheduling. Reduces redundant module rebuilds by making dependency edges explicit. Some projects see regressions due to the overhead of dependency scanning, so benchmark before and after enabling.
- **Risk:** Medium -- test thoroughly; currently experimental for Swift targets.

### Cross-Target Consistency

These checks find settings differences between targets that cause redundant build work.

#### Project-Level vs Target-Level Overrides

Build-affecting settings should be set at the project level unless a target has a specific reason to override. Unnecessary per-target overrides cause confusion and can silently create module variants.

Settings to check for project-level consistency:

- `SWIFT_COMPILATION_MODE`
- `SWIFT_OPTIMIZATION_LEVEL`
- `ONLY_ACTIVE_ARCH`
- `DEBUG_INFORMATION_FORMAT`

#### Module Variant Duplication

When multiple targets import the same SPM package but compile with different Swift compiler options, the build system produces separate module variants for each combination. This inflates `SwiftEmitModule` task counts.

Check for drift in:

- `SWIFT_OPTIMIZATION_LEVEL`
- `SWIFT_COMPILATION_MODE`
- `OTHER_SWIFT_FLAGS`
- Target-level build settings that override project defaults

#### Out of Scope

Do **not** flag the following as build-performance issues:

- `SWIFT_STRICT_CONCURRENCY` -- language migration choice
- `SWIFT_UPCOMING_FEATURE_*` -- language migration choice
- `SWIFT_APPROACHABLE_CONCURRENCY` -- language migration choice
- `SWIFT_ACTIVE_COMPILATION_CONDITIONS` values beyond `DEBUG` (e.g., `WIDGETS`, `APPCLIP`) -- intentional per-target customization

### Checklist Output Format

When reporting results, use this structure:

```markdown
### Debug Configuration
- [x] `SWIFT_COMPILATION_MODE`: `singlefile` (recommended: `singlefile`)
- [ ] `DEBUG_INFORMATION_FORMAT`: `dwarf-with-dsym` (recommended: `dwarf`)
- [x] `SWIFT_OPTIMIZATION_LEVEL`: `-Onone` (recommended: `-Onone`)
...

### Release Configuration
- [x] `SWIFT_COMPILATION_MODE`: `wholemodule` (recommended: `wholemodule`)
...

### General (All Configurations)
- [ ] `COMPILATION_CACHE_ENABLE_CACHING`: `NO` (recommended: `YES`)
...

### Cross-Target Consistency
- [x] All targets inherit `SWIFT_OPTIMIZATION_LEVEL` from project level
- [ ] `OTHER_SWIFT_FLAGS` differs between Stock Analyzer and StockAnalyzerClip
...
```

## Recommendation Format (Full Reference)

All optimization steps should report recommendations in a shared structure so findings can be merged and prioritized cleanly.

### Required Fields

Each recommendation should include:

- `title`
- `wait_time_impact` -- plain-language statement of expected wall-clock impact, e.g. "Expected to reduce your clean build by ~3s", "Reduces parallel compile work but unlikely to reduce build wait time", or "Impact on wait time is uncertain -- re-benchmark to confirm"
- `actionability` -- classifies how fixable the issue is from the project (see values below)
- `category`
- `observed_evidence`
- `estimated_impact`
- `confidence`
- `approval_required`
- `benchmark_verification_status`

#### Actionability Values

Every recommendation must include an `actionability` classification:

- `repo-local` -- Fix lives entirely in project files, source code, or local configuration. The developer can apply it without side effects outside the repo.
- `package-manager` -- Requires CocoaPods or SPM configuration changes that may have broad side effects (e.g., linkage mode, dependency restructuring). These should be benchmarked before and after.
- `xcode-behavior` -- Observed cost is driven by Xcode internals and is not suppressible from the project. Report the finding for awareness but do not promise a fix.
- `upstream` -- Requires changes in a third-party dependency or external tool. The developer cannot fix it locally.

### Suggested Optional Fields

- `scope`
- `affected_files`
- `affected_targets`
- `affected_packages`
- `implementation_notes`
- `risk_level`

### JSON Example

```json
{
  "recommendations": [
    {
      "title": "Guard a release-only symbol upload script",
      "wait_time_impact": "Expected to reduce your incremental build by approximately 6 seconds.",
      "actionability": "repo-local",
      "category": "project",
      "observed_evidence": [
        "Incremental builds spend 6.3 seconds in a run script phase.",
        "The script runs for Debug builds even though the output is only needed in Release."
      ],
      "estimated_impact": "High incremental-build improvement",
      "confidence": "High",
      "approval_required": true,
      "benchmark_verification_status": "Not yet verified",
      "scope": "Target build phase",
      "risk_level": "Low"
    }
  ]
}
```

### Markdown Rendering Guidance

When rendering for human review, preserve the same field order:

1. title
2. wait-time impact
3. actionability
4. observed evidence
5. estimated impact
6. confidence
7. approval required
8. benchmark verification status

That makes it easier for the developer to approve or reject specific items quickly.

### Verification Status Values

Recommended values:

- `Not yet verified`
- `Queued for verification`
- `Verified improvement`
- `No measurable improvement`
- `Inconclusive due to benchmark noise`
