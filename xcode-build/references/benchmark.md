# Benchmark Reference

Full content of the former standalone `xcode-build-benchmark` skill, merged in as a reference for [xcode-build](../SKILL.md). Use this when you need to produce a repeatable Xcode build baseline before anyone tries to optimize build times, or when a developer just wants clean/incremental build numbers without a full optimization pass.

## Core Rules

- Measure before recommending changes.
- Capture clean and incremental builds separately.
- Keep the command, destination, configuration, scheme, and warm-up rules consistent across runs.
- Write a timestamped JSON artifact to `.build-benchmark/`.
- Do not change project files as part of benchmarking.

## Inputs To Collect

Confirm or infer:

- workspace or project path
- scheme
- configuration
- destination
- whether the user wants simulator or device numbers
- whether a custom `DerivedData` path is needed

If the project has both clean-build and incremental-build pain, benchmark both. That is the default.

## Worktree Considerations

When benchmarking inside a git worktree, SPM packages with `exclude:` paths that reference gitignored directories (e.g., `__Snapshots__`) will cause `xcodebuild -resolvePackageDependencies` to crash. Create those missing directories before running any builds.

## Default Workflow

1. Normalize the build command and note every flag that affects caching or module reuse.
2. Run one warm-up build if needed to validate that the command succeeds.
3. Run 3 clean builds.
4. If `COMPILATION_CACHE_ENABLE_CACHING = YES` is detected, run 3 cached clean builds. These measure clean build time with a warm compilation cache -- the realistic scenario for branch switching, pulling changes, or Clean Build Folder. The script handles this automatically by building once to warm the cache, then deleting DerivedData (but not the compilation cache) before each measured run. Pass `--no-cached-clean` to skip.
5. Run 3 zero-change builds (build immediately after a successful build with no edits). This measures the fixed overhead floor: dependency computation, project description transfer, build description creation, script phases, codesigning, and validation. A zero-change build that takes more than a few seconds indicates avoidable per-build overhead. Use the default `benchmark_builds.py` invocation (no `--touch-file` flag).
6. Optionally run 3 incremental builds with a file touch to measure a real edit-rebuild loop. Use `--touch-file path/to/SomeFile.swift` to touch a representative source file before each build.
7. Save the raw results and summary into `.build-benchmark/`.
8. Report medians and spread, not just the single fastest run.

## Preferred Command Path

Use the shared helper when possible:

```bash
python3 scripts/benchmark_builds.py \
  --workspace App.xcworkspace \
  --scheme MyApp \
  --configuration Debug \
  --destination "platform=iOS Simulator,name=iPhone 16" \
  --output-dir .build-benchmark
```

If you cannot use the helper script, run equivalent `xcodebuild` commands with `-showBuildTimingSummary` and preserve the raw output.

## Required Output

Return:

- clean build median, min, max
- cached clean build median, min, max (when COMPILATION_CACHE_ENABLE_CACHING is enabled)
- zero-change build median, min, max (fixed overhead floor)
- incremental build median, min, max (if `--touch-file` was used)
- biggest timing-summary categories
- environment details that could affect comparisons
- path to the saved artifact

If results are noisy, say so and recommend rerunning under calmer conditions.

## When To Stop

Stop after measurement if the user only asked for benchmarking. If they want optimization guidance, hand off the artifact to the relevant specialist using the same project context:

- Compile hotspots and type-checking -- see [compilation-analysis.md](compilation-analysis.md)
- Project configuration, build settings, schemes -- see [project-analysis.md](project-analysis.md)
- Package graph and build plugins -- see [`spm-build-analysis`](../../spm-build-analysis/SKILL.md) (standalone sibling skill)
- Full end-to-end orchestration -- see the main [xcode-build SKILL.md](../SKILL.md)

## Benchmarking Workflow (Full Operational Contract)

Use this section when you need the full operational contract for collecting Xcode build measurements.

### Goal

Produce a benchmark artifact that another skill (or reference playbook) can trust without rerunning the same setup discovery.

### Benchmark Contract

- Measure both clean and incremental builds unless the user narrows the scope.
- Use the same scheme, configuration, destination, and command flags for all measured runs.
- Record the exact command and any environment overrides.
- Keep clean and incremental runs in separate arrays in the artifact.
- Save wall-clock timing plus any parsed timing-summary categories.

### Suggested Run Counts

- Clean builds: 3 measured runs
- Incremental builds: 3 measured runs
- Warm-up: 0 to 1 validation run, excluded from the summary unless the user explicitly wants it included

### Clean Build Rules

- Clear build products with `xcodebuild clean` or an equivalent clean-build-folder step before each measured clean run.
- Do not change scheme, destination, or configuration between runs.
- If the command fails, store the failure and stop rather than mixing failed and successful runs.

### Incremental Build Rules

- Use the same build command after a successful baseline build.
- Do not clean between incremental runs.
- If the user wants edit-loop benchmarking, note the file change strategy explicitly in the artifact.
- If there are no source edits between runs, label the result as no-edit incremental timing.

### What To Capture

At minimum, keep:

- timestamp
- host machine info if available
- Xcode version if available
- workspace or project path
- scheme, configuration, destination
- exact `xcodebuild` command
- duration per run
- success or failure
- parsed timing-summary categories
- notes on warm-up behavior or unusual noise

### Reporting Guidance

Use medians for the headline number. Also include:

- min and max
- range
- category totals from the timing summary
- obvious outliers or instability

### Handoff Expectations

The next optimization step should be able to answer:

- Is the main problem clean, incremental, or both?
- Which build categories dominate time?
- Which command produced the evidence?
- Is the baseline trustworthy enough to compare before and after changes?

## Benchmark Artifacts (Shared Artifact Format)

All reference playbooks in this skill should treat `.build-benchmark/` as the canonical location for measured build evidence.

### Goals

- Keep build measurements reproducible.
- Make clean and incremental build data easy to compare.
- Preserve enough context for later specialist analysis without rerunning the benchmark.

### Wall-Clock vs Cumulative Task Time

The `duration_seconds` field on each run and the `median_seconds` in the summary represent **wall-clock time** -- how long the developer actually waits. This is the primary success metric.

The `timing_summary_categories` are **aggregated task times** parsed from Xcode's Build Timing Summary. Because Xcode runs many tasks in parallel across CPU cores, these totals typically exceed the wall-clock duration. A large cumulative `SwiftCompile` value is diagnostic evidence of compiler workload, not proof that compilation is blocking the build. Always compare category totals against the wall-clock median before concluding that a category is a bottleneck.

### File Layout

Recommended outputs:

- `.build-benchmark/<timestamp>-<scheme>.json`
- `.build-benchmark/<timestamp>-<scheme>-clean-1.log`
- `.build-benchmark/<timestamp>-<scheme>-clean-2.log`
- `.build-benchmark/<timestamp>-<scheme>-clean-3.log`
- `.build-benchmark/<timestamp>-<scheme>-cached-clean-1.log` (when COMPILATION_CACHE_ENABLE_CACHING is enabled)
- `.build-benchmark/<timestamp>-<scheme>-cached-clean-2.log`
- `.build-benchmark/<timestamp>-<scheme>-cached-clean-3.log`
- `.build-benchmark/<timestamp>-<scheme>-incremental-1.log`
- `.build-benchmark/<timestamp>-<scheme>-incremental-2.log`
- `.build-benchmark/<timestamp>-<scheme>-incremental-3.log`

Use an ISO-like UTC timestamp without spaces so the files sort naturally.

### Artifact Requirements

Each JSON artifact should include:

- schema version
- creation timestamp
- project context
- environment details when available
- the normalized build command
- separate `clean` and `incremental` run arrays
- summary statistics for each build type
- parsed timing-summary categories
- free-form notes for caveats or noise

### Clean, Cached Clean, And Incremental Separation

Do not merge different build type measurements into a single list. They answer different questions:

- **Clean builds** show full build-system, package, and module setup cost with a cold compilation cache.
- **Cached clean builds** show clean build cost when the compilation cache is warm. This is the realistic scenario for branch switching, pulling changes, or Clean Build Folder. Only present when `COMPILATION_CACHE_ENABLE_CACHING = YES` is detected.
- **Incremental builds** show edit-loop productivity and script or cache invalidation problems.

### Raw Logs

Store raw `xcodebuild` output beside the JSON artifact whenever possible. That allows later analysis steps to:

- re-parse timing summaries
- inspect failed builds
- search for long type-check warnings
- correlate build-system phases with recommendations

### Measurement Caveats

#### COMPILATION_CACHE_ENABLE_CACHING

`COMPILATION_CACHE_ENABLE_CACHING = YES` stores compiled artifacts in a system-managed cache outside DerivedData so that repeated compilations of identical inputs are served from cache. The standard clean-build benchmark (`xcodebuild clean` between runs) may add overhead from cache population without showing the corresponding cache-hit benefit.

The benchmark script automatically detects `COMPILATION_CACHE_ENABLE_CACHING = YES` and runs a **cached clean** benchmark phase. This phase:

1. Builds once to warm the compilation cache.
2. Deletes DerivedData (but not the compilation cache) before each measured run.
3. Rebuilds, measuring the cache-hit clean build time.

The cached clean metric captures the realistic developer experience: branch switching, pulling changes, and Clean Build Folder. Use the cached clean median as the primary comparison metric when evaluating `COMPILATION_CACHE_ENABLE_CACHING` impact.

To skip this phase, pass `--no-cached-clean`.

#### First-Run Variance

The first clean build after the warmup cycle often runs 20-40% slower than subsequent clean builds due to cold OS-level caches (disk I/O, dynamic linker cache, etc.). The benchmark script mitigates this by running a warmup clean+build cycle before measured runs. If variance between the first and later clean runs is still high, prefer the median or min over the mean, and note the variance in the artifact's `notes` field.

### Shared Consumer Expectations

Anything reading a benchmark artifact should be able to identify:

- what was measured
- how it was measured
- whether the run succeeded
- whether the results are stable enough to compare

For the authoritative field-level schema, see [../schemas/build-benchmark.schema.json](../schemas/build-benchmark.schema.json).
