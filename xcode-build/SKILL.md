---
name: xcode-build
description: Optimize Xcode build times end-to-end -- benchmark clean/incremental/cached-clean builds, diagnose slow compilation and Swift type-checking, audit project configuration/build settings/schemes/script phases, and apply approved fixes with re-benchmarking to prove the win. This is the single entrypoint for ALL Xcode build-speed work -- use it whenever a developer wants a full build optimization pass, asks to speed up Xcode builds, wants to benchmark or measure build performance, asks "how long do my builds take" or wants before/after numbers, reports slow compilation, expensive type-checking, long CompileSwiftSources/SwiftEmitModule/"Planning Swift module" tasks, wants a build settings or scheme/script-phase audit, or has an approved optimization plan ready to implement. Always reach for this skill first for Xcode build-performance work rather than improvising ad hoc fixes.
---

# Xcode Build

Use this skill as the recommend-first entrypoint for end-to-end Xcode build optimization work. It is the single entry point for benchmarking, diagnosing, and fixing slow Xcode builds -- it routes to bundled reference playbooks for each specialty rather than requiring separate skills.

## Non-Negotiable Rules

- Wall-clock build time (how long the developer waits) is the primary success metric. Every recommendation must state its expected impact on the developer's actual wait time.
- Start in recommendation mode.
- Benchmark before making changes.
- Do not modify project files, source files, packages, or scripts without explicit developer approval.
- Preserve the evidence trail for every recommendation.
- Re-benchmark after approved changes and report the wall-clock delta.

## Two-Phase Workflow

The workflow is designed as two distinct phases separated by developer review.

### Phase 1 -- Analyze (recommend-only)

Run this phase in agent mode because the agent needs to execute builds, run benchmark scripts, write benchmark artifacts, and generate the optimization report. However, treat Phase 1 as **recommend-only**: do not modify any project files, source files, packages, or build settings. The only files the agent creates during this phase are benchmark artifacts and the optimization plan inside `.build-benchmark/`.

1. Collect the build target context: workspace or project, scheme, configuration, destination, and current pain point. When both `.xcworkspace` and `.xcodeproj` exist, prefer `.xcodeproj` unless the workspace contains sub-projects required for the build. Workspaces that reference external projects may fail if those projects are not checked out.
2. Establish a baseline benchmark if no fresh one exists -- follow [references/benchmark.md](references/benchmark.md) for the full measurement contract (clean, cached-clean, zero-change, and incremental runs). The benchmark script auto-detects `COMPILATION_CACHE_ENABLE_CACHING = YES` and includes cached clean builds that measure the realistic developer experience (warm cache). If the build fails to compile, check `git log` for a recent buildable commit. When working in a worktree, cherry-picking a targeted build fix from a feature branch is acceptable to reach a buildable state. If SPM packages reference gitignored directories in their `exclude:` paths (e.g., `__Snapshots__`), create those directories before building -- worktrees do not contain gitignored content and `xcodebuild -resolvePackageDependencies` will crash otherwise.
3. Verify the benchmark artifact has non-empty `timing_summary_categories`. If empty, the timing summary parser may have failed -- re-parse the raw logs or inspect them manually. If `COMPILATION_CACHE_ENABLE_CACHING` is enabled, also verify the artifact includes `cached_clean` runs.
   - **Benchmark confidence check**: For each build type (clean, cached clean, incremental), compare the min and max values. If the spread (max - min) exceeds 20% of the median, flag the benchmark as having high variance and recommend running additional repetitions (5+ runs) before drawing conclusions. High variance makes it difficult to distinguish real improvements from noise. After applying changes, only claim an improvement if the post-change median falls outside the baseline's min-max range.
4. If incremental builds are the primary pain point and Xcode 16.4+ is available, recommend the developer enable **Task Backtraces** (Scheme Editor > Build tab > Build Debugging > "Task Backtraces"). This reveals why each task re-ran, which is critical for diagnosing unexpected replanning or input invalidation. Include any Task Backtrace evidence in the analysis.
5. Determine whether compile tasks are likely blocking wall-clock progress or just consuming parallel CPU time. Compare the sum of all timing-summary category seconds against the wall-clock median: if the sum is 2x+ the median, most work is parallelized and compile hotspot fixes are unlikely to reduce wait time. If `SwiftCompile`, `CompileC`, `SwiftEmitModule`, or `Planning Swift module` dominate the timing summary **and** appear likely to be on the critical path, run `diagnose_compilation.py` to capture type-checking hotspots. If they are parallelized, still run diagnostics but label findings as "parallel efficiency improvements" rather than "build time improvements."
6. Run the specialist analyses that fit the evidence:
   - Source-level compile hotspots and Swift type-checking -- see [references/compilation-analysis.md](references/compilation-analysis.md)
   - Project configuration, build settings, schemes, script phases -- see [references/project-analysis.md](references/project-analysis.md)
   - Package graph and build plugin overhead -- see [`spm-build-analysis`](../spm-build-analysis/SKILL.md) (a separate, standalone skill; read its SKILL.md and apply its workflow to the same project context)
7. Merge findings into a single prioritized improvement plan.
8. Generate the markdown optimization report using `generate_optimization_report.py` and save it to `.build-benchmark/optimization-plan.md`. This report includes the build settings audit, timing analysis, prioritized recommendations, and an approval checklist.
9. Stop and present the plan to the developer for review.

The developer reviews `.build-benchmark/optimization-plan.md`, checks the approval boxes for the recommendations they want implemented, and then triggers phase 2.

### Phase 2 -- Execute and verify (agent mode)

Run this phase in agent mode after the developer has reviewed and approved recommendations from the plan. Apply all implementation work by following [references/fixer.md](references/fixer.md).

10. Read `.build-benchmark/optimization-plan.md` and identify the approved items from the approval checklist.
11. Apply the approved plan per [references/fixer.md](references/fixer.md): apply each approved change, verify compilation, and re-benchmark.
12. Append verification results to the optimization plan: post-change medians, absolute and percentage deltas, and confidence notes.
13. Report before and after results, plus any remaining follow-up opportunities.

## Prioritization Rules

The goal is to reduce how long the developer waits for builds to finish.

1. Identify the developer's primary pain (clean build, incremental build, or both) and the measured wall-clock median.
2. Determine what is likely **blocking** wall-clock progress:
   - If the sum of all timing-summary category seconds is 2x+ the wall-clock median, most work is parallelized. Compile hotspot fixes are unlikely to reduce wait time.
   - If a single serial category (e.g. `PhaseScriptExecution`, `CompileAssetCatalog`, `CodeSign`) accounts for a large fraction of wall-clock, that is the real bottleneck.
   - If `Planning Swift module` or `SwiftEmitModule` dominates incremental builds, the cause is likely invalidation or module size, not individual file compile speed.
3. Rank recommendations by likely wall-time savings, not cumulative task reduction.
4. Source-level compile fixes should not outrank project/graph/configuration fixes unless evidence suggests they are on the critical path.

Prefer changes that are measurable, reversible, and low-risk.

## Recommendation Impact Language

Every recommendation presented to the developer must include one of these impact statements:

- "Expected to reduce your [clean/incremental] build by approximately X seconds."
- "Reduces parallel compile work but is unlikely to reduce your build wait time because other tasks take equally long."
- "Impact on wait time is uncertain -- re-benchmark after applying to confirm."
- "No wait-time improvement expected. The benefit is [deterministic builds / faster branch switching / reduced CI cost]."
- For COMPILATION_CACHE_ENABLE_CACHING specifically: "Measured 5-14% faster clean builds across tested projects. The benefit compounds in real workflows where the cache persists between builds -- branch switching, pulling changes, and CI with persistent DerivedData."

Never quote cumulative task-time savings as the headline impact. If a change reduces 5 seconds of parallel compile work but another equally long task still runs, the developer's wait time does not change.

## Approval Gate

Before implementing anything, present a short approval list that includes:

- recommendation name
- expected wait-time impact (using the impact language above)
- evidence summary
- affected files or settings
- whether the change is low, medium, or high risk

Wait for explicit developer approval.

## Post-Approval Execution

After approval, apply changes by following [references/fixer.md](references/fixer.md):

- implement only the approved items
- apply changes atomically and keep them scoped
- note any deviations from the original recommendation plan
- re-benchmark with the same benchmark contract

## Final Report

Lead with the wall-clock result in plain language, e.g.: "Your clean build now takes 82s (was 86s) -- 4s faster." Then include:

- baseline clean and incremental wall-clock medians
- post-change clean and incremental wall-clock medians
- absolute and percentage wall-clock deltas
- what changed
- what was intentionally left unchanged
- confidence notes if noise prevents a strong conclusion -- if benchmark variance is high (min-to-max spread exceeds 20% of median), say so explicitly rather than presenting noisy numbers as definitive improvements or regressions
- if cumulative task metrics improved but wall-clock did not, say plainly: "Compiler workload decreased but build wait time did not improve. This is expected when Xcode runs these tasks in parallel with other equally long work."
- a ready-to-paste community results row and a link to open a PR (see the report template)

## Preferred Command Paths

### Benchmark

```bash
python3 scripts/benchmark_builds.py \
  --project App.xcodeproj \
  --scheme MyApp \
  --configuration Debug \
  --destination "platform=iOS Simulator,name=iPhone 16" \
  --output-dir .build-benchmark
```

For macOS apps use `--destination "platform=macOS"`. For watchOS use `--destination "platform=watchOS Simulator,name=Apple Watch Series 10"`. For tvOS use `--destination "platform=tvOS Simulator,name=Apple TV"`. Omit `--destination` to use the scheme's default.

To measure real incremental builds (file-touched rebuild) instead of zero-change builds, add `--touch-file path/to/SomeFile.swift`.

### Compilation Diagnostics

```bash
python3 scripts/diagnose_compilation.py \
  --project App.xcodeproj \
  --scheme MyApp \
  --configuration Debug \
  --destination "platform=iOS Simulator,name=iPhone 16" \
  --threshold 100 \
  --output-dir .build-benchmark
```

### Optimization Report

```bash
python3 scripts/generate_optimization_report.py \
  --benchmark .build-benchmark/<artifact>.json \
  --project-path App.xcodeproj \
  --diagnostics .build-benchmark/<diagnostics>.json \
  --output .build-benchmark/optimization-plan.md
```

## Specialist Reference Playbooks

These are loaded on demand -- open the relevant file only when that phase of work is active:

- [references/benchmark.md](references/benchmark.md) -- full measurement contract: inputs to collect, clean/cached-clean/zero-change/incremental run rules, required output, and the benchmark artifact format.
- [references/compilation-analysis.md](references/compilation-analysis.md) -- source-level compile hotspot and Swift type-checking analysis: what to inspect, diagnostic flags, Apple-derived checks, and source citations.
- [references/project-analysis.md](references/project-analysis.md) -- project/target/scheme configuration audit: build settings best practices, script phase checks, zero-change overhead analysis, CocoaPods migration guidance.
- [references/fixer.md](references/fixer.md) -- how to apply approved fixes: build settings, script phases, source-level patterns, SPM restructuring, regression evaluation, and the execution report format.
- [references/orchestration-report-template.md](references/orchestration-report-template.md) -- the full `.build-benchmark/optimization-plan.md` report template that `generate_optimization_report.py` produces automatically.

The shared recommendation structure (fields every recommendation must include) is embedded in each of references/compilation-analysis.md, references/project-analysis.md, and references/fixer.md. Build settings best practices are embedded in references/project-analysis.md and references/fixer.md.
