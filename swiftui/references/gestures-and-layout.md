# Gesture Composition and Adaptive Layout

This is the advanced-interaction reference: composing multiple gestures (simultaneous / sequenced / exclusive), and building layouts that adapt to container size and platform traits rather than to assumed device classes. This material comes from a source dedicated specifically to this sub-domain — treat it as the authority for gesture and adaptive-layout questions, not a footnote to general view/performance guidance.

## Part 1 — Gesture Composition

### Decision Tree

```
What interaction do you need?
- Single tap/click? -> Button (preferred) or TapGesture
- Drag/pan? -> DragGesture
- Hold before action? -> LongPressGesture
- Pinch to zoom? -> MagnificationGesture
- Two-finger rotate? -> RotationGesture

Multiple gestures together?
- Both at same time? -> .simultaneously
- One after another? -> .sequenced
- One OR the other? -> .exclusively
```

**Composition order matters.** `.simultaneously` and `.sequenced` have different trigger timing — swapping them silently changes behavior. Understand gesture semantics before combining them; don't guess.

### GestureState vs State

| Use Case | Type | Why |
|----------|------|-----|
| Temporary feedback | `@GestureState` | Auto-resets when gesture ends |
| Final committed value | `@State` | Persists after gesture |

### Pattern 1: Draggable View

```swift
struct DraggableCard: View {
    @GestureState private var dragOffset = CGSize.zero  // Temporary
    @State private var position = CGSize.zero           // Permanent

    var body: some View {
        RoundedRectangle(cornerRadius: 12)
            .offset(x: position.width + dragOffset.width,
                    y: position.height + dragOffset.height)
            .gesture(
                DragGesture()
                    .updating($dragOffset) { value, state, _ in
                        state = value.translation
                    }
                    .onEnded { value in
                        withAnimation(.spring()) {
                            position.width += value.translation.width
                            position.height += value.translation.height
                        }
                    }
            )
    }
}
```

### Pattern 2: Simultaneous Gestures

```swift
// Drag AND pinch-zoom at the same time
.gesture(
    DragGesture()
        .updating($dragOffset) { value, state, _ in state = value.translation }
        .simultaneously(with:
            MagnificationGesture()
                .updating($scale) { value, state, _ in state = value.magnification }
        )
)
```

### Pattern 3: Sequenced Gestures

```swift
// Long press THEN drag (like iOS Home Screen reordering)
LongPressGesture(minimumDuration: 0.5)
    .onEnded { _ in isEditing = true }
    .sequenced(before:
        DragGesture()
            .updating($dragOffset) { value, state, _ in state = value.translation }
    )
```

### Pattern 4: Exclusive Gestures

```swift
// Double-tap OR single-tap (not both)
TapGesture(count: 2)
    .onEnded { zoom() }
    .exclusively(before:
        TapGesture(count: 1)
            .onEnded { select() }
    )
```

### Common Pitfalls

**Using @State instead of @GestureState:**
```swift
// WRONG - offset stays at last value
@State private var offset = CGSize.zero

// CORRECT - auto-resets when gesture ends
@GestureState private var offset = CGSize.zero
```

**Gesture blocks ScrollView:**
```swift
// WRONG - blocks scrolling
.gesture(DragGesture())

// CORRECT - allows both
.simultaneousGesture(DragGesture())
```

**Using TapGesture instead of Button:**
```swift
// WRONG - no accessibility
Text("Submit").onTapGesture { }

// CORRECT - proper semantics
Button("Submit") { }
```

If `onTapGesture()` must be used (e.g. you specifically need tap location or tap count), add `.accessibilityAddTraits(.isButton)` so VoiceOver still announces it correctly.

### Gesture Accessibility

```swift
Image("slider")
    .gesture(DragGesture().onChanged { ... })
    .accessibilityAdjustableAction { direction in
        switch direction {
        case .increment: volume += 5
        case .decrement: volume -= 5
        @unknown default: break
        }
    }
```

## Part 2 — Adaptive Layout

### Core Principle

Respond to your container, not assumptions about the device. Your layout should work if Apple ships a new device or multitasking mode tomorrow.

### Decision Tree

```
"I need my layout to adapt..."

TO AVAILABLE SPACE:
- Pick best-fitting variant? -> ViewThatFits
- Animated H/V switch? -> AnyLayout + condition
- Read size for calculations? -> onGeometryChange (iOS 16+)

TO PLATFORM TRAITS:
- Compact vs Regular width? -> horizontalSizeClass
- Accessibility text size? -> dynamicTypeSize.isAccessibilitySize
```

### Pattern 1: ViewThatFits

SwiftUI picks the first variant that fits.

```swift
ViewThatFits {
    HStack { Image(systemName: "star"); Text("Favorite"); Button("Add") { } }
    VStack { Image(systemName: "star"); Text("Favorite"); Button("Add") { } }
}
```

**`ViewThatFits` is over-used in practice.** It remeasures on every view change, which is expensive for anything beyond a couple of static variants. For animated H/V switches driven by a known condition (like size class), use `AnyLayout` instead — reserve `ViewThatFits` for genuinely static variant selection.

### Pattern 2: AnyLayout

Animated transitions between layouts.

```swift
@Environment(\.horizontalSizeClass) var sizeClass

var layout: AnyLayout {
    sizeClass == .compact
        ? AnyLayout(VStackLayout(spacing: 12))
        : AnyLayout(HStackLayout(spacing: 20))
}

var body: some View {
    layout { content }
        .animation(.default, value: sizeClass)
}
```

### Pattern 3: onGeometryChange

Read dimensions without `GeometryReader`'s greedy-expansion side effects.

```swift
@State private var columnCount = 2

LazyVGrid(columns: Array(repeating: GridItem(.flexible()), count: columnCount)) {
    ForEach(items) { ItemView(item: $0) }
}
.onGeometryChange(for: Int.self) { proxy in
    max(1, Int(proxy.size.width / 150))
} action: { columnCount = $0 }
```

**Watch for feedback loops.** Reading geometry can change geometry, which triggers another update, which reads geometry again — a cycle. Only write to state that doesn't itself affect the measured geometry, or debounce/guard the write.

### Size Class on iPad

| Configuration | Horizontal |
|--------------|------------|
| Full screen | `.regular` |
| 50% Split View | `.regular` |
| 33% Split View | `.compact` |
| Slide Over | `.compact` |

**Key insight**: size class only goes `.compact` on iPad at ~33% width.

### Anti-Patterns

**Device orientation observer:**
```swift
// WRONG - reports device, not window
UIDevice.current.orientation

// CORRECT - read actual dimensions
.onGeometryChange(for: Bool.self) { $0.size.width > $0.size.height }
```

**Screen bounds:**
```swift
// WRONG - returns full screen, ignores multitasking/Stage Manager
UIScreen.main.bounds.width

// CORRECT - read container size
.onGeometryChange(for: CGFloat.self) { $0.size.width }
```

**Device model checks:**
```swift
// WRONG - fails in multitasking
if UIDevice.current.userInterfaceIdiom == .pad { }

// CORRECT - respond to space
@Environment(\.horizontalSizeClass) var sizeClass
```

**Unconstrained GeometryReader:**
```swift
// WRONG - expands greedily
GeometryReader { geo in Text("\(geo.size)") }

// CORRECT - constrain it
GeometryReader { geo in Text("\(geo.size)") }
    .frame(height: 44)
```

### iOS 26 Changes

- `UIRequiresFullScreen` deprecated
- Free-form window resizing
- `NavigationSplitView` auto-adapts columns
- Remove full-screen-only restriction from Info.plist where it no longer applies

## Common Mistakes (both parts)

1. **Gesture composition order matters** — see above; `.simultaneously`/`.sequenced`/`.exclusively` are not interchangeable.
2. **`ViewThatFits` over-used** — prefer `AnyLayout` for animated switches driven by a known condition.
3. **`onGeometryChange` triggering unnecessary updates** — guard against read-triggers-write-triggers-read cycles.
4. **Ignoring view body optimization in gesture/layout-heavy views** — expensive calculations inside a frequently-re-measured `body` compound with layout remeasurement cost. Move calculations to properties or models; profile with Instruments before optimizing prematurely (see `references/performance-patterns.md`).
