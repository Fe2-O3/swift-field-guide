---
name: swiftui
description: Use for anything SwiftUI on iOS or macOS -- writing, reviewing, or refactoring views; state management (@State/@Observable/@Bindable/environment); gesture composition (simultaneous/sequenced/exclusive); adaptive layout (ViewThatFits/AnyLayout/size classes); architecture choice (MVVM vs TCA vs vanilla); navigation and presentation (NavigationStack, sheets, Inspector); modern API adoption and Liquid Glass; accessibility and HIG compliance; performance and maintainability review; or Instruments `.trace` capture/analysis for hangs, hitches, and excessive view updates. Reach for this before improvising SwiftUI guidance from memory -- it consolidates five prior overlapping skills and is more current and more complete than ad hoc knowledge.
---

# SwiftUI

This skill merges five previously-overlapping SwiftUI skills into one. It covers the full span from "write this view" to "review this PR" to "why is this list janky" to "compose these two gestures" to "should this be MVVM or TCA." Consult the reference files aggressively — they hold the actual patterns, code, and decision detail; this file is a router plus the handful of rules that apply everywhere.

## Operating Rules

- Consult `references/latest-apis.md` at the start of every task to avoid deprecated APIs.
- Prefer native SwiftUI APIs over UIKit/AppKit bridging unless bridging is genuinely necessary (`references/uikit-interop.md`).
- Focus on correctness and performance; don't force a specific architecture (MVVM, TCA, vanilla) — help the user choose one deliberately instead (`references/architecture.md`).
- Encourage separating business logic from views for testability, without mandating exactly how.
- Follow Apple's Human Interface Guidelines and API design idioms (`references/design-and-hig.md`).
- Only adopt Liquid Glass when explicitly requested (`references/liquid-glass.md`).
- Present performance optimizations as suggestions, not blanket requirements — profile before optimizing prematurely.
- Use `#available` gating with sensible fallbacks for version-specific APIs.
- iOS 26 is a reasonable default deployment target for new apps; target Swift 6.2+ with modern concurrency (`references/swift-language-conventions.md`).
- Don't introduce third-party frameworks without asking first.
- This skill's guidance is opinionated by design, but where two merged sources genuinely disagreed, the reference files say so explicitly rather than presenting a false consensus — see the "Flagged conflict" notes in `references/state-management.md` and `references/view-structure.md`.

## Task Workflow

### Review existing SwiftUI code
- Read the code under review and identify which topics apply.
- Flag deprecated APIs (compare against `references/latest-apis.md`).
- Run the Topic Router below for each relevant topic.
- Validate `#available` gating and fallback paths for iOS 26+ features.
- For a **comprehensive** review (not a single-topic look), follow the full process, output format, and hygiene checklist in `references/review-checklist.md` instead of ad hoc commentary.

### Improve existing SwiftUI code
- Audit the current implementation against the Topic Router topics.
- Replace deprecated APIs with modern equivalents from `references/latest-apis.md`.
- Refactor hot paths to reduce unnecessary state updates (`references/performance-patterns.md`).
- Extract complex view bodies into separate subviews, not `@ViewBuilder` computed properties (`references/view-structure.md` — flagged-conflict note explains why).
- Suggest image downsampling when `UIImage(data:)` is encountered (optional, `references/image-optimization.md`).

### Implement a new SwiftUI feature
- Design data flow first: identify owned vs. injected state (`references/state-management.md`).
- Structure views for optimal diffing (extract subviews early).
- Pick an architecture deliberately if the feature is non-trivial (`references/architecture.md`).
- Apply correct animation patterns (implicit vs. explicit, transitions — `references/animation-basics.md`, `references/animation-transitions.md`, `references/animation-advanced.md`).
- Use `Button` for all tappable elements; add accessibility grouping and labels (`references/accessibility-patterns.md`).
- If the feature needs multi-gesture interaction or must adapt across window sizes, go straight to `references/gestures-and-layout.md` — it's the dedicated reference for both.
- Gate version-specific APIs with `#available` and provide fallbacks.

### Record a new Instruments trace
Trigger when the user asks to "record a trace", "profile the app", "capture a session", etc. Full reference: `references/trace-recording.md`.

1. **Confirm target** — attach to a running app, launch an app, or record all processes? If unstated, ask. List connected devices when useful:
   ```bash
   python3 "${SKILL_DIR}/scripts/record_trace.py" --list-devices
   ```
2. **Pick a template based on target kind** — the `SwiftUI` template populates the SwiftUI lane on any **real device** (physical iOS/iPadOS device or the host Mac). The exception is the **iOS Simulator**, where the SwiftUI lane comes back empty — switch to `--template "Time Profiler"` there (still gives Time Profiler + Hangs + Animation Hitches). `--list-devices`: `simulators` kind -> `Time Profiler`; `devices` kind -> default `SwiftUI`. Full decision table in `references/trace-recording.md`.
3. **Start the recording.** For agent-driven sessions where the user says "I'll tell you when I'm done", start in the background with a stop-file:
   ```bash
   python3 "${SKILL_DIR}/scripts/record_trace.py" \
       --device "<name|udid>" --attach "<AppName>" \
       --stop-file /tmp/stop-trace --output ~/Desktop/session.trace
   ```
   For interactive sessions, tell the user to press Ctrl+C when done.
4. **Signal stop** — `touch /tmp/stop-trace` once the user says they're finished exercising the app. The script cleanly SIGINTs xctrace and waits up to 60s for finalisation.
5. **Analyse** the resulting trace (flows into "Trace-driven improvement" below).

For a lighter-weight, no-file-required alternative for a quick look, see the built-in Xcode "SwiftUI Instrument" workflow (Cmd-I) documented in `references/performance-patterns.md`.

### Trace-driven improvement (Instruments `.trace` provided)
Trigger whenever the user's request references a `.trace` file. A target SwiftUI source file is **optional** — if given, cite specific lines; if not, recommend where to look based on view names and symbols the trace already reveals.

Full reference: `references/trace-analysis.md`. Summary of the composition pattern:

1. **Scope the analysis.** "focus on X / after X / between X and Y / during X" -> resolve a window first (step 2). No scoping cue -> analyse the whole trace.
2. **Resolve a window (only if scoped).**
   ```bash
   python3 "${SKILL_DIR}/scripts/analyze_trace.py" --trace <path> \
       --list-logs --log-message-contains "loaded feed" --log-limit 5
   # or
   python3 "${SKILL_DIR}/scripts/analyze_trace.py" --trace <path> \
       --list-signposts --signpost-name-contains "ImageDecode"
   ```
   Both accept `--window START_MS:END_MS` to scope discovery; build a window like `--window 10400:11700`.
3. **Run the main analysis:**
   ```bash
   python3 "${SKILL_DIR}/scripts/analyze_trace.py" --trace <path> \
       --json-only --top 10 [--window START_MS:END_MS]
   ```
4. **Interpret with `references/trace-analysis.md`** — key diagnostics: `main_running_coverage_pct` (<25% = blocked; >=75% = CPU-bound); `swiftui-causes.top_sources` reveals *why* updates keep happening.
5. **When a specific view is expensive, ask who's invalidating it** — `--fanin-for "<view name>"`.
6. **Optionally ground in source.** Match view/symbol names against the file if one was given; otherwise recommend which files to open.
7. **Return a prioritised plan** citing evidence and routing each recommendation to a Topic Router reference.
8. Only edit code if the user asked for edits.

## Decision Trees

### Gesture Composition
- Both gestures at the same time? -> `.simultaneously`
- One must complete before the next? -> `.sequenced`
- Only one should win? -> `.exclusively`

Full patterns, `@GestureState` vs `@State`, and pitfalls: `references/gestures-and-layout.md`.

### Layout Adaptation
- Pick the best-fitting static variant? -> `ViewThatFits`
- Animated H/V switch driven by a known condition? -> `AnyLayout`
- Need actual container dimensions? -> `onGeometryChange`

Full patterns, size-class table, and anti-patterns: `references/gestures-and-layout.md`.

### Architecture Selection
- Small app, comfortable with Apple's own patterns? -> `@Observable` + State-as-Bridge
- Complex presentation logic, team knows MVVM? -> MVVM with `@Observable` view models
- Rigorous testability, large team? -> TCA (defer to the `composable-architecture` skill for TCA specifics)

Full decision tree, State-as-Bridge pattern, and anti-patterns: `references/architecture.md`.

## Topic Router

| Topic | Reference |
|-------|-----------|
| **Read first, every task** | `references/latest-apis.md` -- deprecated-to-modern API transitions (iOS 15+ through 26+) |
| State management | `references/state-management.md` |
| View composition / structure | `references/view-structure.md` |
| Architecture (MVVM / TCA / vanilla) | `references/architecture.md` |
| **Gestures & adaptive layout** | `references/gestures-and-layout.md` |
| Performance | `references/performance-patterns.md` |
| Lists and ForEach | `references/list-patterns.md` |
| Layout (non-gesture) | `references/layout-best-practices.md` |
| Sheets, navigation, Inspector | `references/sheet-navigation-patterns.md` |
| ScrollView | `references/scroll-patterns.md` |
| Focus management | `references/focus-patterns.md` |
| Async patterns (`.task`, `.refreshable`, cancellation) | `references/async-patterns.md` |
| UIKit/AppKit interop | `references/uikit-interop.md` |
| Migrating iOS 16 -> 17+ code | `references/migration-guide.md` |
| Animations (basics) | `references/animation-basics.md` |
| Animations (transitions) | `references/animation-transitions.md` |
| Animations (advanced / `@Animatable`) | `references/animation-advanced.md` |
| Accessibility | `references/accessibility-patterns.md` |
| Design & Human Interface Guidelines | `references/design-and-hig.md` |
| Swift Charts | `references/charts.md` |
| Charts accessibility | `references/charts-accessibility.md` |
| Image optimization | `references/image-optimization.md` |
| Liquid Glass (iOS 26+) | `references/liquid-glass.md` |
| macOS scenes | `references/macos-scenes.md` |
| macOS window styling | `references/macos-window-styling.md` |
| macOS views | `references/macos-views.md` |
| Text patterns | `references/text-patterns.md` |
| Swift language conventions | `references/swift-language-conventions.md` |
| Previews | `references/previews.md` |
| Comprehensive code review (process + output format) | `references/review-checklist.md` |
| Instruments trace analysis | `references/trace-analysis.md` |
| Instruments trace recording | `references/trace-recording.md` |

## Correctness Checklist

These are hard rules — violations are always bugs:

- [ ] `@State` properties are `private`
- [ ] `@Binding` only where a child modifies parent state
- [ ] Passed values never declared as `@State` or `@StateObject` (they ignore updates)
- [ ] `@StateObject` for view-owned objects; `@ObservedObject` for injected (legacy, pre-iOS 17)
- [ ] iOS 17+: `@State` with `@Observable`; `@Bindable` for injected observables needing bindings
- [ ] `ForEach` uses stable identity (never `.indices` for dynamic content)
- [ ] Constant number of views per `ForEach` element
- [ ] `.animation(_:value:)` always includes the `value` parameter
- [ ] `@FocusState` properties are `private`
- [ ] No redundant `@FocusState` writes inside tap gesture handlers on `.focusable()` views
- [ ] iOS 26+ APIs gated with `#available` and fallback provided
- [ ] `import Charts` present in files using chart types
- [ ] Previews use self-contained mock data; no dependency on live services or network
- [ ] `@ObservationIgnored` on every property-wrapper property inside an `@Observable` class (`@AppStorage`, `@SceneStorage`, `@Query`, etc.) — and don't assume `@AppStorage` there still triggers view updates (see `references/state-management.md`)
- [ ] `@AppStorage`/`UserDefaults` never used for secrets — Keychain only
- [ ] `navigationDestination(for:)` and `NavigationLink(destination:)` are never mixed in the same navigation hierarchy

For the fuller review process (output format, prioritized findings, hygiene pass), see `references/review-checklist.md`.

## References

- `references/latest-apis.md` — **Read first for every task.** Deprecated-to-modern API transitions (iOS 15+ through iOS 26+), plus a flat list of additional modern-API rules.
- `references/api-refresh.md` — Maintenance workflow for refreshing `latest-apis.md` after a new Xcode/iOS release: scan Apple's docs via the Sosumi MCP (see `references/api-refresh-scan-manifest.md`) for new deprecations, then update the file.
- `references/state-management.md` — Property wrappers, data flow, `@Observable` migration, `@AppStorage`-in-`@Observable` caveat, numeric `TextField` binding, SwiftData quick notes.
- `references/view-structure.md` — View extraction, container patterns, `@ViewBuilder`, the extract-to-struct-vs-computed-property conflict, UIKit interop essentials.
- `references/architecture.md` — MVVM vs TCA vs vanilla decision tree, State-as-Bridge, worked MVVM example, anti-patterns.
- `references/gestures-and-layout.md` — Gesture composition (simultaneous/sequenced/exclusive), `@GestureState`, `ViewThatFits`/`AnyLayout`/`onGeometryChange`, size classes, iOS 26 layout changes.
- `references/performance-patterns.md` — Hot-path optimization, update control, `_logChanges()`, Instruments 26 SwiftUI template protocol, debounced search, pagination.
- `references/list-patterns.md` — ForEach identity, Table (iOS 16+), inline filtering pitfalls.
- `references/layout-best-practices.md` — Layout patterns, GeometryReader alternatives.
- `references/sheet-navigation-patterns.md` — Sheets, NavigationStack/NavigationSplitView, Inspector, navigation coordinator pattern.
- `references/scroll-patterns.md` — ScrollViewReader, programmatic scrolling.
- `references/focus-patterns.md` — Focus state, focusable views, focused values, default focus, common pitfalls.
- `references/async-patterns.md` — `.task`/`.task(id:)`/`.refreshable`/background tasks, nested-Task cancellation.
- `references/uikit-interop.md` — `UIViewRepresentable`/`UIViewControllerRepresentable` worked examples, delegate-cycle leaks.
- `references/migration-guide.md` — iOS 16 -> 17+ migration checklist with before/after.
- `references/accessibility-patterns.md` — VoiceOver, Dynamic Type, grouping, traits, named actions, Reduce Motion, Differentiate Without Color.
- `references/design-and-hig.md` — Uniform design tokens, HIG compliance, tap targets, system styling.
- `references/animation-basics.md` — Implicit/explicit animations, timing, performance.
- `references/animation-transitions.md` — View transitions, `matchedGeometryEffect`, `Animatable`.
- `references/animation-advanced.md` — Phase/keyframe animations (iOS 17+), `@Animatable` macro (iOS 26+).
- `references/charts.md` — Swift Charts marks, axes, selection, styling, Chart3D (iOS 26+).
- `references/charts-accessibility.md` — Charts VoiceOver, Audio Graph, fallback strategies.
- `references/image-optimization.md` — AsyncImage, downsampling, caching.
- `references/liquid-glass.md` — iOS 26+ Liquid Glass effects and fallback patterns.
- `references/macos-scenes.md` — Settings, MenuBarExtra, WindowGroup, multi-window.
- `references/macos-window-styling.md` — Toolbar styles, window sizing, Commands.
- `references/macos-views.md` — HSplitView, Table, PasteButton, AppKit interop.
- `references/text-patterns.md` — Text initializer selection, verbatim vs. localized.
- `references/swift-language-conventions.md` — Swift-native API choices and concurrency conventions for SwiftUI codebases.
- `references/previews.md` — `#Preview` macro, `@Previewable` (iOS 18+), preview traits, mock data patterns.
- `references/review-checklist.md` — Full comprehensive-review process, output format, Core Instructions, hygiene checklist.
- `references/trace-analysis.md` — Parse Instruments `.trace` files via `scripts/analyze_trace.py`; interpret main-thread coverage, high-severity SwiftUI updates, hitch narratives, map findings back to source.
- `references/trace-recording.md` — Record a new trace via `scripts/record_trace.py`: attach, launch, or capture a manually-stopped session; supports stop-file for agent-driven flows.
