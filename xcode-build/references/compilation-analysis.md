# Compilation Analysis Reference

Full content of the former standalone `xcode-compilation-analyzer` skill, merged in as a reference for [xcode-build](../SKILL.md). Use this when compile time, not just general project configuration, looks like the bottleneck.

## Core Rules

- Start from evidence, ideally a recent `.build-benchmark/` artifact or raw timing-summary output.
- Prefer analysis-only compiler flags over persistent project edits during investigation.
- Rank findings by expected **wall-clock** impact, not cumulative compile-time impact. When compile tasks are heavily parallelized (sum of compile categories >> wall-clock median), note that fixing individual hotspots may improve parallel efficiency without reducing build wait time.
- When the evidence points to parallelized work rather than serial bottlenecks, label recommendations as "Reduces compiler workload (parallel)" rather than "Reduces build time."
- Do not edit source or build settings without explicit developer approval.

## What To Inspect

- `Build Timing Summary` output from clean and incremental builds
- long-running `CompileSwiftSources` or per-file compilation tasks
- `SwiftEmitModule` time -- can reach 60s+ after a single-line change in large modules; if it dominates incremental builds, the module is likely too large or macro-heavy
- `Planning Swift module` time -- if this category is disproportionately large in incremental builds (up to 30s per module), it signals unexpected input invalidation or macro-related rebuild cascading
- ad hoc runs with:
  - `-Xfrontend -warn-long-expression-type-checking=<ms>`
  - `-Xfrontend -warn-long-function-bodies=<ms>`
- deeper diagnostic flags for thorough investigation:
  - `-Xfrontend -debug-time-compilation` -- per-file compile times to rank the slowest files
  - `-Xfrontend -debug-time-function-bodies` -- per-function compile times (unfiltered, complements the threshold-based warning flags)
  - `-Xswiftc -driver-time-compilation` -- driver-level timing to isolate driver overhead
  - `-Xfrontend -stats-output-dir <path>` -- detailed compiler statistics (JSON) per compilation unit for root-cause analysis
- mixed Swift and Objective-C surfaces that increase bridging work

## Analysis Workflow

1. Identify whether the main issue is broad compilation volume or a few extreme hotspots.
2. Parse timing-summary categories and rank the biggest compile contributors.
3. Run the diagnostics script to surface type-checking hotspots:
   ```bash
   python3 scripts/diagnose_compilation.py \
     --project App.xcodeproj \
     --scheme MyApp \
     --configuration Debug \
     --destination "platform=iOS Simulator,name=iPhone 16" \
     --threshold 100 \
     --output-dir .build-benchmark
   ```
   This produces a ranked list of functions and expressions that exceed the millisecond threshold. Use the diagnostics artifact alongside source inspection to focus on the most expensive files first.
4. Map the evidence to a concrete recommendation list.
5. Separate code-level suggestions from project-level or module-level suggestions.

## Apple-Derived Checks

Look for these patterns first:

- missing explicit type information in expensive expressions
- complex chained or nested expressions that are hard to type-check
- delegate properties typed as `AnyObject` instead of a concrete protocol
- oversized Objective-C bridging headers or generated Swift-to-Objective-C surfaces
- header imports that skip framework qualification and miss module-cache reuse
- classes missing `final` that are never subclassed
- overly broad access control (`public`/`open`) on internal-only symbols
- monolithic SwiftUI `body` properties that should be decomposed into subviews
- long method chains or closures without intermediate type annotations

## Reporting Format

For each recommendation, include:

- observed evidence
- likely affected file or module
- expected wait-time impact (e.g. "Expected to reduce your clean build by ~2s" or "Reduces parallel compile work but unlikely to reduce build wait time")
- confidence
- whether approval is required before applying it

If the evidence points to project configuration instead of source, hand off to [project-analysis.md](project-analysis.md) using the same project context.

## Preferred Tactics

- Suggest ad hoc flag injection through the build command before recommending persistent build-setting changes.
- Prefer narrowing giant view builders, closures, or result-builder expressions into smaller typed units.
- Recommend explicit imports and protocol typing when they reduce compiler search space.
- Call out when mixed-language boundaries are the real issue rather than Swift syntax alone.

## Compilation Checks (Full Reference)

Use this section when a build benchmark shows compilation dominating build time.

### Primary Evidence Sources

- `xcodebuild -showBuildTimingSummary`
- build log compile tasks
- `-warn-long-function-bodies`
- `-warn-long-expression-type-checking`
- `-debug-time-compilation` (per-file compile time ranking)
- `-debug-time-function-bodies` (unfiltered per-function timing)
- `-driver-time-compilation` (driver overhead)
- `-stats-output-dir` (detailed compiler statistics as JSON)

### Triage Questions

1. Is one file or expression dominating compile time?
2. Is the issue mostly Swift type-checking, mixed-language bridging, or header import churn?
3. Are multiple files in the same module paying the same module-setup cost repeatedly?
4. Is `SwiftEmitModule` disproportionately large for any target? If a single-line change triggers 60s+ of module emission, the target is likely too large or heavily macro-dependent.
5. Does `Planning Swift module` dominate incremental builds? If modules are replanned but no compiles are scheduled, build inputs are being invalidated unexpectedly.

### Checklist

#### Explicit typing

- Add explicit property or local variable types when initialization expressions are complex.
- Prefer intermediate typed variables over one giant inferred expression.

#### Expression simplification

- Break long chains into smaller expressions.
- Split complex result-builder code into smaller helpers or subviews.
- Replace nested ternaries or overloaded generic chains with simpler steps.

#### Delegate typing

- Avoid `AnyObject?` or overly generic delegate surfaces.
- Prefer a named delegate protocol so the compiler has a narrower lookup space.

#### Objective-C and Swift bridging

- Keep the Objective-C bridging header narrow.
- Move internal-only Objective-C declarations out of the bridging surface.
- Mark Swift members `private` when they do not need Objective-C visibility.

#### Framework-qualified imports

- Prefer `#import <Framework/Header.h>` or module imports when a module map exists.
- Watch for textual includes that defeat module-cache reuse.

#### Access control and dispatch optimization

- Mark classes not intended for subclassing as `final`. This eliminates virtual dispatch overhead and lets the compiler de-virtualize method calls, reducing both compile and runtime cost.
- Use `private` or `fileprivate` for properties and methods not used outside their declaration or file. Narrower visibility reduces the compiler's symbol search space.
- Prefer `internal` (the default) over `public` unless the symbol genuinely crosses module boundaries. Wider access forces the compiler to consider more call sites.

#### Value types over reference types

- Prefer `struct` and `enum` over `class` when reference semantics are not needed. Value types are simpler for the compiler to reason about and do not require vtable dispatch.
- When a class exists solely to group data without identity semantics, convert it to a struct.

#### SwiftUI view decomposition

- Extract subviews into dedicated `struct View` types instead of using `@ViewBuilder` helper properties. Separate structs reduce the type-checker scope per `body` property.
- Break monolithic `body` properties (roughly 50+ lines) into smaller composed subviews. Large result-builder bodies are among the most expensive expressions to type-check.
- Avoid deeply nested `Group`/`VStack`/`HStack` hierarchies within a single body.

#### Closure and chain patterns

- Avoid long method chains like `.map().flatMap().filter().reduce()` without intermediate type annotations. Each link in the chain multiplies the type-checker's candidate set.
- Break complex closures into named functions with explicit parameter and return types.
- Add explicit return types to closures passed to generic functions so the compiler does not need to infer them from context.

#### Generic constraint complexity

- Minimize deeply nested generic constraints (e.g., `where T: Collection, T.Element: Comparable, T.Element.SubSequence: ...`). Each additional constraint widens the compiler's search space.
- Use type aliases to flatten complex generic stacks into readable names.
- Prefer `some Protocol` (opaque return types) over unconstrained generics when the concrete type does not need to be visible to callers.

#### Module emission and planning overhead

- Check `SwiftEmitModule` time in the Build Timing Summary. Large modules with many public symbols take longer to emit, and this cost is paid on every incremental build that touches the module.
- If `SwiftEmitModule` exceeds compile time for the same target, the module's public API surface may be unnecessarily wide -- narrow access control or split the module.
- Check `Planning Swift module` time. If it is significant in incremental builds, escalate to [project-analysis.md](project-analysis.md) to investigate unexpected input invalidation or misconfigured scripts.

#### Precompiled and prefix headers

- For mixed-language projects with large Objective-C codebases, verify that prefix headers are not bloated with unnecessary imports. Every import in a prefix header is parsed for every translation unit.
- Migrate away from prefix headers toward explicit module imports where possible.

### Recommendation Heuristics

- High impact: repeated type-check warnings in a hot module, giant bridging headers, or a few files dominating compile time.
- Medium impact: several moderate hotspots in result builders or overloaded generic code.
- Low impact: isolated warnings without measurable benchmark impact.

### Escalation Guidance

Hand findings to [project-analysis.md](project-analysis.md) when:

- build scripts dominate instead of compilation
- module reuse is blocked by project settings
- target structure or explicit-module settings appear to be the real bottleneck
- `Planning Swift module` overhead points to input invalidation or script-related causes rather than source complexity

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

This section stores the external sources that reports and recommendations should cite consistently.

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
