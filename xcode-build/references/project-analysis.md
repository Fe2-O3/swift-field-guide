# Project Analysis Reference

Full content of the former standalone `xcode-project-analyzer` skill, merged in as a reference for [xcode-build](../SKILL.md). Use this for project- and target-level build inefficiencies that are unlikely to be solved by source edits alone.

## Core Rules

- Recommendation-first by default.
- Require explicit approval before changing project files, schemes, or build settings.
- Prefer measured findings tied to timing summaries, build logs, or project configuration evidence.
- Distinguish debug-only pain from release-only pain.

## What To Review

- scheme build order and target dependencies
- debug vs release build settings against the [build settings best practices](#build-settings-best-practices-full-reference) below
- run script phases and dependency-analysis settings
- derived-data churn or obviously invalidating custom steps
- opportunities for parallelization
- explicit module dependency settings and module-map readiness
- "Planning Swift module" time in the Build Timing Summary -- if it dominates incremental builds, suspect unexpected input modification or macro-related invalidation
- asset catalog compilation time, especially in targets with large or numerous catalogs
- `ExtractAppIntentsMetadata` time in the Build Timing Summary -- if this phase consumes significant time, record it as `xcode-behavior` (report the cost and impact, but do not suggest a repo-local optimization unless there is explicit Apple guidance)
- zero-change build overhead -- if a no-op rebuild exceeds a few seconds, investigate fixed-cost phases (script execution, codesign, validation, CopySwiftLibs)
- CocoaPods usage -- if a `Podfile` or `Pods.xcodeproj` exists, CocoaPods is deprecated; recommend migrating to SPM and do not attempt CocoaPods-specific optimizations (see [Project Audit Checklist](#project-audit-checklist-full-reference) below)
- Task Backtraces (Xcode 16.4+: Scheme Editor > Build > Build Debugging) to diagnose why tasks re-run unexpectedly in incremental builds

## Build Settings Best Practices Audit

Every project audit should include a build settings checklist comparing the project's Debug and Release configurations against the recommended values in [Build Settings Best Practices](#build-settings-best-practices-full-reference) below. Present results using checkmark/cross indicators (`[x]`/`[ ]`). The scope is strictly build performance -- do not flag language-migration settings like `SWIFT_STRICT_CONCURRENCY` or `SWIFT_UPCOMING_FEATURE_*`.

## Apple-Derived Checks

Review these items in every audit:

- target dependencies are accurate and not missing or inflated
- schemes build in `Dependency Order`
- run scripts declare inputs and outputs
- `.xcfilelist` files are used when scripts have many inputs or outputs
- `DEFINES_MODULE` is enabled where custom frameworks or libraries should expose module maps
- headers are self-contained enough for module-map use
- explicit module dependency settings are consistent for targets that should share modules

## Typical Wins

- skip debug-time scripts that only matter in release
- add missing script guards or dependency-analysis metadata
- remove accidental serial bottlenecks in schemes
- align build settings that cause unnecessary module variants
- fix stale project structure that forces broader rebuilds than necessary
- identify linters or formatters that touch file timestamps without changing content, silently invalidating build inputs and forcing module replanning
- split large asset catalogs into separate resource bundles across targets to parallelize compilation
- use Task Backtraces to pinpoint the exact input change that triggers unnecessary incremental work

## Reporting Format

For each issue, include:

- evidence
- likely scope
- why it affects clean builds, incremental builds, or both
- estimated impact
- approval requirement

If the evidence points to package graph or build plugins, hand off to [`spm-build-analysis`](../../spm-build-analysis/SKILL.md) (a separate, standalone skill; read its SKILL.md and apply its workflow to the same project context).

## Project Audit Checklist (Full Reference)

Use this section when reviewing build-system configuration rather than source-level compile behavior.

### Target And Scheme Checks

- Confirm target dependencies are explicit and accurate.
- Remove dependencies that no longer reflect real build requirements.
- Ensure the scheme builds targets in `Dependency Order`.
- Look for oversized or monolithic targets that block parallel work.

### Build Script Checks

- Does each script need to run during incremental builds?
- Are input and output files declared?
- Should inputs and outputs be moved into `.xcfilelist` files?
- Can the script skip debug builds, simulator builds, or unchanged inputs?
- Would the script become parallelizable if dependency analysis were declared correctly?
- Could a misconfigured linter or formatter script be touching file timestamps without changing content? This silently invalidates build inputs and forces replanning of every module.

### Build Planning And Incremental Overhead Checks

- Check whether "Planning Swift module" appears as a significant category in the Build Timing Summary. In projects with many modules this step can take up to 30s per module, sometimes exceeding clean build time.
- If modules are replanned but no compiles are scheduled, suspect unexpected input modification (see Build Script Checks above).
- Enable **Task Backtraces** (Xcode 16.4+) to diagnose why tasks re-run in incremental builds: Scheme Editor > Build tab > Build Debugging > enable "Task Backtraces." Expanding tasks in the build log will show a backtrace explaining what input change triggered the re-run.
- Measure zero-change incremental build time as a baseline. Even with no source changes, builds incur fixed overhead: compute dependencies, send project description to build service, create build description, run script phases, codesign, and validate. If this baseline exceeds a few seconds, investigate each contributor.
- Check whether codesigning and validation run on every build even when the build output has not changed.

### Zero-Change Build Overhead Analysis

When a zero-change build (no edits, immediate rebuild) takes more than a few seconds, the overhead comes from fixed-cost phases rather than compilation. Investigate these categories in the Build Timing Summary:

- **PhaseScriptExecution**: Script phases with `alwaysOutOfDate = 1` or missing input/output declarations run on every build regardless of changes. Linters, formatters, and upload scripts are common offenders.
- **CodeSign**: Codesigning runs on every build for the app target and any embedded frameworks. Time scales with the number of signed binaries.
- **ValidateEmbeddedBinary**: Validates embedded binaries against the host app's provisioning profile. Runs unconditionally.
- **CopySwiftLibs**: Copies Swift standard libraries into the app bundle. Runs even when nothing changed.
- **RegisterWithLaunchServices**: Registers the built app with Launch Services. Fast but present in every build.
- **ProcessInfoPlistFile**: Re-processes Info.plist files. Time scales with the number of targets.
- **ExtractAppIntentsMetadata**: Extracts App Intents metadata from all targets. This phase is driven by Xcode and runs across all targets including CocoaPods and SwiftPM dependencies, not just first-party targets. If the project does not use App Intents, the work is unnecessary overhead, but it is not cleanly suppressible from project-level build settings alone. Classify findings about this phase as `xcode-behavior` actionability -- report the measured cost for awareness but do not promise a repo-local fix.

A zero-change build above 5 seconds on Apple Silicon typically indicates script phase overhead or an excessive number of targets requiring codesign and validation passes.

### Asset Catalog Checks

- Asset catalog compilation (`CompileAssetCatalog`) is single-threaded per target. Multiple catalogs within the same target compile sequentially in a single process.
- If asset catalog compilation appears as a significant timing category, recommend splitting assets into separate resource bundles across separate targets to enable parallel compilation.
- Asset catalog compilation is not cacheable by the Xcode compilation cache (`CompileAssetCatalogVariant` is non-cacheable).
- Check whether asset catalogs rebuild during incremental builds even when no assets changed. If so, investigate whether build inputs for the catalog step are being invalidated unexpectedly.

### Build Setting Checks

Audit project-level and target-level settings against [Build Settings Best Practices](#build-settings-best-practices-full-reference) below. Present results as a checklist with `[x]`/`[ ]` indicators.

Key settings to verify:

- `SWIFT_COMPILATION_MODE` -- `singlefile` for Debug, `wholemodule` for Release
- `SWIFT_OPTIMIZATION_LEVEL` -- `-Onone` for Debug, `-O` or `-Osize` for Release
- `ONLY_ACTIVE_ARCH` -- `YES` for Debug, `NO` for Release
- `DEBUG_INFORMATION_FORMAT` -- `dwarf` for Debug, `dwarf-with-dsym` for Release
- `GCC_OPTIMIZATION_LEVEL` -- `0` for Debug, `s` for Release
- `ENABLE_TESTABILITY` -- `YES` for Debug, `NO` for Release
- `COMPILATION_CACHE_ENABLE_CACHING` -- recommended `YES` for all configurations; caches repeated compilations during branch switching and clean builds
- `EAGER_LINKING` -- recommended `YES` for Debug; starts linking before all compilation finishes
- `SWIFT_USE_INTEGRATED_DRIVER` -- recommended `YES`; uses the integrated driver for better scheduling
- `CLANG_ENABLE_MODULES` -- recommended `YES`; caches module maps on disk for C/ObjC

Do not flag language-migration settings (`SWIFT_STRICT_CONCURRENCY`, `SWIFT_UPCOMING_FEATURE_*`) as build performance issues.

### Module And Header Checks

- `DEFINES_MODULE` is enabled for custom frameworks that should benefit from module maps.
- Public headers are self-contained enough to compile as a module.
- Import statements use framework-qualified imports where available.
- targets that should share built modules use consistent options

### Explicit Module Dependency Checks

- Check whether explicit modules are enabled or expected in the current Xcode version and Swift mode.
- Look for repeated module builds caused by configuration drift.
- Compare preprocessor macros or other build options across sibling targets that import the same modules.

### CocoaPods Projects

CocoaPods is deprecated. Do not attempt CocoaPods-specific build optimizations such as linkage mode changes (`use_frameworks! :linkage => :static`), `COCOAPODS_PARALLEL_CODE_SIGN`, or Podfile tweaks. These are unreliable and frequently regress build times.

When a project uses CocoaPods (presence of `Podfile`, `Pods/`, or a `Pods.xcodeproj`), recommend migrating to Swift Package Manager as the highest-impact long-term improvement. SPM advantages for build time:

- **Compilation caching**: `COMPILATION_CACHE_ENABLE_CACHING` works with SPM targets out of the box, delivering cumulative benefits across branch switching, pulling changes, and CI.
- **Better build parallelism**: SPM targets build in parallel based on the dependency graph without the overhead of a separate Pods project.
- **No xcconfig regeneration**: CocoaPods regenerates xcconfigs and its own project file on every `pod install`. SPM resolution is lighter and its outputs integrate natively.
- **Native Xcode integration**: No separate `Pods.xcodeproj`, no workspace stitching, and full support for modern Xcode features like explicit modules.

Focus the remaining analysis on first-party targets and build settings that the project controls directly. Do not audit or recommend changes to `Pods.xcodeproj` or the Podfile.

### Recommendation Prioritization

Qualify every estimated impact with wall-clock framing. High-priority items should be those likely to reduce the developer's actual wait time, not just cumulative task totals. If the impact on wait time is uncertain, say so.

- High: serial script bottlenecks, missing dependency metadata, configuration drift causing redundant module builds, excessive "Planning Swift module" time, or scripts silently invalidating build inputs.
- Medium: stale target structure, noncritical scripts running too often, slow asset catalog compilation blocking the critical path, unnecessary codesigning on unchanged output, or significant `ExtractAppIntentsMetadata` time in projects without App Intents.
- Low: settings cleanup without strong evidence of current impact.

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

## Source Citations (Full Reference)

This file stores the external sources that reports and recommendations should cite consistently.

### Apple: Improving the speed of incremental builds

Source:

- <https://developer.apple.com/documentation/xcode/improving-the-speed-of-incremental-builds>

Key takeaways:

- Measure first with `Build With Timing Summary` or `xcodebuild -showBuildTimingSummary`.
- Accurate target dependencies improve correctness and parallelism.
- Run scripts should declare inputs and outputs so Xcode can skip unnecessary work.
- `.xcfilelist` files are appropriate when scripts have many inputs or outputs.
- Custom frameworks and libraries benefit from module maps, typically by enabling `DEFINES_MODULE`.
- Module reuse is strongest when related sources compile with consistent options.
- Breaking monolithic targets into better-scoped modules can reduce unnecessary rebuilds.

### Apple: Improving build efficiency with good coding practices

Source:

- <https://developer.apple.com/documentation/xcode/improving-build-efficiency-with-good-coding-practices>

Key takeaways:

- Use framework-qualified imports when module maps are available.
- Keep Objective-C bridging surfaces narrow.
- Prefer explicit type information when inference becomes expensive.
- Use explicit delegate protocols instead of overly generic delegate types.
- Simplify complex expressions that are hard for the compiler to type-check.

### Apple: Building your project with explicit module dependencies

Source:

- <https://developer.apple.com/documentation/xcode/building-your-project-with-explicit-module-dependencies>

Key takeaways:

- Explicit module builds make module work visible in the build log and improve scheduling.
- Repeated builds of the same module often point to avoidable module variants.
- Inconsistent build options across targets can force duplicate module builds.
- Timing summaries can reveal option drift that prevents module reuse.

### SwiftLee: Build performance analysis for speeding up Xcode builds

Source:

- <https://www.avanderlee.com/optimization/analysing-build-performance-xcode/>

Key takeaways:

- Clean and incremental builds should both be measured because they reveal different problems.
- Build Timeline and Build Timing Summary are practical starting points for build optimization.
- Build scripts often produce large incremental-build wins when guarded correctly.
- `-warn-long-function-bodies` and `-warn-long-expression-type-checking` help surface compile hotspots.
- Typical debug and release build setting mismatches are worth auditing, especially in older projects.

### Apple: Xcode Release Notes -- Compilation Caching

Source:

- Xcode Release Notes (149700201)

Key takeaways:

- Compilation caching is an opt-in feature for Swift and C-family languages.
- It caches prior compilation results and reuses them when the same source inputs are recompiled.
- Branch switching and clean builds benefit the most.
- Can be enabled via the "Enable Compilation Caching" build setting or per-user project settings.

### Apple: Demystify explicitly built modules (WWDC24)

Source:

- <https://developer.apple.com/videos/play/wwdc2024/10171/>

Key takeaways:

- Explains how explicitly built modules divide compilation into scan, module build, and source compile stages.
- Unrelated modules build in parallel, improving CPU utilization.
- Module variant duplication is a key bottleneck -- uniform compiler options across targets prevent it.
- The build log shows each module as a discrete task, making it easier to diagnose scheduling issues.

### Swift Compile-Time Best Practices

Well-known Swift language patterns that reduce type-checker workload during compilation:

- Mark classes `final` when they are not intended for subclassing. This eliminates dynamic dispatch overhead and allows the compiler to de-virtualize method calls.
- Restrict access control to the narrowest useful scope (`private`, `fileprivate`). Fewer visible symbols reduce the compiler's search space during type resolution.
- Prefer value types (`struct`, `enum`) over `class` when reference semantics are not needed. Value types are simpler for the compiler to reason about.
- Break long method chains (`.map().flatMap().filter()`) into intermediate `let` bindings with explicit type annotations. Even simple-looking chains can take seconds to type-check.
- Provide explicit return types on closures passed to generic functions, especially in SwiftUI result-builder contexts.
- Decompose large SwiftUI `body` properties into smaller extracted subviews. Each subview narrows the scope of the result-builder expression the type-checker must resolve.

### Bitrise: Demystifying Explicitly Built Modules for Xcode

Source:

- <https://bitrise.io/blog/post/demystifying-explicitly-built-modules-for-xcode>

Key takeaways:

- Explicit module builds give `xcodebuild` visibility into smaller compilation tasks for better parallelism.
- Enabled by default for C/Objective-C in Xcode 16+; experimental for Swift.
- Minimizing module variants by aligning build options is the primary optimization lever.
- Some projects see regressions from dependency scanning overhead -- benchmark before and after.

### Bitrise: Xcode Compilation Cache FAQ

Source:

- <https://docs.bitrise.io/en/bitrise-build-cache/build-cache-for-xcode/xcode-compilation-cache-faq.html>

Key takeaways:

- Granular caching is controlled by `SWIFT_ENABLE_COMPILE_CACHE` and `CLANG_ENABLE_COMPILE_CACHE`, under the umbrella `COMPILATION_CACHE_ENABLE_CACHING` setting.
- Non-cacheable tasks include `CompileStoryboard`, `CompileXIB`, `CompileAssetCatalogVariant`, `PhaseScriptExecution`, `DataModelCompile`, `CopyPNGFile`, `GenerateDSYMFile`, and `Ld`.
- SPM dependencies are not yet cacheable as of Xcode 26 beta.

### RocketSim Docs: Build Insights

Sources:

- <https://www.rocketsim.app/docs/features/build-insights/build-insights/>
- <https://www.rocketsim.app/docs/features/build-insights/team-build-insights/>

Key takeaways:

- RocketSim automatically tracks clean vs incremental builds over time without build scripts.
- It reports build counts, duration trends, and percentile-based metrics such as p75 and p95.
- Team Build Insights adds machine, Xcode, and macOS comparisons for cross-team visibility.
- This repository is best positioned as the point-in-time analyze-and-improve toolkit, while RocketSim is the monitor-over-time companion.

### Swift Forums: Slow incremental builds because of planning swift module

Source:

- <https://forums.swift.org/t/slow-incremental-builds-because-of-planning-swift-module/84803>

Key takeaways:

- "Planning Swift module" can dominate incremental builds (up to 30s per module), sometimes exceeding clean build time.
- Replanning every module without scheduling compiles is a sign that build inputs are being modified unexpectedly (e.g., a misconfigured linter touching file timestamps).
- Enable **Task Backtraces** (Xcode 16.4+: Scheme Editor > Build > Build Debugging) to see why each task re-ran in an incremental build.
- Heavy Swift macro usage (e.g., TCA / swift-syntax) can cause trivial changes to cascade into near-full rebuilds.
- `swift-syntax` builds universally (all architectures) when no prebuilt binary is available, adding significant overhead.
- `SwiftEmitModule` can take 60s+ after a single-line change in large modules.
- Asset catalog compilation is single-threaded per target; splitting assets into separate bundles across targets enables parallel compilation.
- Multi-platform targets (e.g., adding watchOS) can cause SPM packages to build 3x (iOS arm64, iOS x86_64, watchOS arm64).
- Zero-change incremental builds still incur ~10s of fixed overhead: compute dependencies, send project description, create build description, script phases, codesigning, and validation.
- Codesigning and validation run even when output has not changed.
