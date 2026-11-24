# Advanced SwiftUI — Internals & Expert APIs

Deep-dive beyond `03-swiftui.md`: identity/diffing, preferences, custom layouts, matched geometry, transitions, symbol effects, focus, drag/drop, representables, transactions, Charts, Liquid Glass, performance.

## 1. View identity & the diffing engine

- SwiftUI identifies views **structurally**: a view's identity = its position in the tree + type + explicit `.id()`. It is NOT object identity (views are values).
- `ForEach` requires stable, unique `id` per element — `Identifiable` conformance or explicit `id:`. Reordering with unstable IDs causes state loss and animation glitches.
- `.id(value)` — forces a NEW identity when value changes → full rebuild of that subtree (state reset). Use for "refresh this view when X changes" semantics.
- Conditional branches (`if/else`, `switch`) create different subtrees — state inside a branch is destroyed when the branch disappears. Keep conditional views at leaf level or hoist state up.
- `@State` lifetime is tied to view identity in the tree. Moving a view in the tree (reordering) preserves state; deleting+recreating loses it.
- `_ConditionalContent`, `_TupleView`, `_VariadicView` — compiler-generated view types for `if/else` and variadic children; they affect diffing granularity.
- **Why "my @State resets"**: the view's position/type changed, or an ancestor rebuilt with different structure. Fix: stable IDs, hoist state to parent/model, avoid `if` around stateful views.

## 2. PreferenceKey — upward data flow

```swift
struct ScrollOffsetKey: PreferenceKey {
  static var defaultValue: CGFloat = 0
  static func reduce(value: inout CGFloat, nextValue: () -> CGFloat) {
    value = nextValue()   // or combine multiple children's values
  }
}

struct OffsetReader: View {
  var body: some View {
    GeometryReader { geo in
      Color.clear.preference(key: ScrollOffsetKey.self, value: geo.frame(in: .named("scroll")).minY)
    }
  }
}

ScrollView {
  LazyVStack { ForEach(items) { ItemView($0) } ; OffsetReader() }
}
.coordinateSpace(name: "scroll")
.onPreferenceChange(ScrollOffsetKey.self) { offset in ... }
```

- Preferences flow **up** the tree (child → ancestor); `onPreferenceChange` observes. Used for scroll offsets, sizes, anchors — the "correct" way vs GeometryReader hacks.
- `reduce` combines values from multiple children — order is bottom-up; choose last/first/max as needed.
- Performance: preference updates trigger ancestor re-eval — keep payloads small (CGFloat/Bool), avoid updating every frame unless needed; pair with `.equatable()` or throttle.
- `Anchor<Label>` preference values + `.anchorPreference` — capture a view's frame/edge as `Anchor<CGRect>` for custom alignment/overlays.

## 3. Coordinate spaces

- `.global` — window/screen coordinates.
- `.local` — the view's own coordinate space (bounds origin).
- `.named("id")` — named space via `.coordinateSpace(name:)` on an ancestor — THE pattern for scroll-relative geometry:
  ```swift
  ScrollView { content }.coordinateSpace(name: "scroll")
  // child: geo.frame(in: .named("scroll"))
  ```
- `visualEffect { content, geometry in }` — per-frame geometry access for scroll effects WITHOUT GeometryReader layout cost.
- `onGeometryChange(for:of:action:)` — observe geometry changes (iOS 16+): `onGeometryChange(for: CGRect.self) { $0.frame(in: .global) } action: { new, old in }`.

## 4. Custom Layout protocol

```swift
struct FlowLayout: Layout {
  var spacing: CGFloat = 8

  func sizeThatFits(proposal: ProposedViewSize, subviews: Subviews, cache: inout ()) -> CGSize {
    let result = arrange(proposal: proposal, subviews: subviews, in: CGRect(origin: .zero, size: proposal.replacingUnspecifiedDimensions().size ?? .zero))
    return result.size
  }
  func placeSubviews(in bounds: CGRect, proposal: ProposedViewSize, subviews: Subviews, cache: inout ()) {
    let result = arrange(proposal: proposal, subviews: subviews, in: bounds)
    for (index, position) in result.positions.enumerated() {
      subviews[index].place(at: position, proposal: .unspecified)
    }
  }
  // cache type can be a class for expensive measurement storage
}
```

- `Layout` requirements: `sizeThatFits(proposal:subviews:cache:)` → CGSize; `placeSubviews(in:proposal:subviews:cache:)`.
- `Subviews` — collection of `LayoutSubview`: `.sizeThatFits(proposal:)`, `.dimensions(in:)`, `.place(at:proposal:)`, `.spacing`, `.layoutPriority`, `.alignmentGuide`.
- `ProposedViewSize` — width/height `ViewDimension?` (.unspecified/.fixed/.flexible ranges); `replacingUnspecifiedDimensions()`.
- `Cache` — associatedtype for caching subview measurements across layout passes (class ref type for mutability).
- `ExplicitAlignment` — guide vertical/horizontal layout; `alignmentGuide(_:computeValue:)` on subviews.
- Use cases: flow layouts, radial layouts, custom stacks, Pinterest-style. `Arrangement` helper exists? — implement manually.
- Invalidatable: `invalidateLayout()` on the Layout instance when config changes.

## 5. matchedGeometryEffect — deep

```swift
@Namespace private var ns

// Source
Circle().matchedGeometryEffect(id: "hero", in: ns)
// or .matchedTransitionSource(id:in:) — iOS 18+ newer API for zoom transitions

// Destination (different branch of if/else or different view)
RoundedRectangle(cornerRadius: 12).matchedGeometryEffect(id: "hero", in: ns)

// With properties control:
.matchedGeometryEffect(id: "hero", in: ns, properties: .frame, isSource: true)
.matchedGeometryEffect(id: "hero", in: ns, properties: .position, anchor: .top, isSource: false)
```

- `properties:` — `.frame`, `.position`, `.size`, `.opacity` — which attributes interpolate.
- `isSource:` — exactly ONE view per namespace+id should be source at a time; both marked → undefined.
- `anchor:` — alignment point for the match.
- Works across navigation pushes, sheets, conditional swaps — the hero-transition API.
- **iOS 18+**: `.navigationTransition(.zoom(sourceID:in:))` on NavigationLink + `.matchedTransitionSource(id:in:)` — system zoom transition (cleaner for navigation).
- Combine with `withAnimation` — the effect animates implicitly when state changes inside animation.

## 6. Transitions & content transitions

```swift
// Insert/remove transitions
if show { DetailView().transition(.asymmetric(insertion: .move(edge: .trailing), removal: .opacity)) }
.transition(.push(from: .bottom))   // built-in: .push, .move, .opacity, .scale, .slide, .offset
.transition(.modifier(active: X(), identity: Y()))  // custom via view modifier

// AnyTransition custom
extension AnyTransition {
  static var blur: AnyTransition {
    .modifier(active: BlurView(radius: 20), identity: BlurView(radius: 0))
  }
}
```

- Transitions fire on **insert/remove** within a container (if/else branches, ForEach add/remove) — NOT on appear of already-present views.
- `.transition` + animation: wrap state change in `withAnimation` or use `.animation(_, value:)`.
- `contentTransition` — content CHANGES (not insert/remove):
  - `.numericText()` — digit roll for counters (iOS 16+)
  - `.numericText(countsDown:)` — countdown variant
  - `.symbolEffect` — SF Symbol transitions
  - `.interpolate` — crossfade-ish
  - `.identity`
  ```swift
  Text("\(count)").contentTransition(.numericText())
  ```
- `symbolEffect` modifiers: `.bounce`, `.pulse`, `.variableColor`, `.appear`, `.disappear`, `.replace` — with `.symbolEffect(.bounce, value: trigger)`.
- `symbolRenderingMode`, `symbolVariant`, `imageScale` — rendering control.

## 7. Focus system

```swift
@FocusState private var focusedField: Field?
enum Field: Hashable { case email, password }

TextField("Email", text: $email).focused($focusedField, equals: .email)
SecureField("Password", text: $pw).focused($focusedField, equals: .password)
Button("Next") { focusedField = .password }
Button("Submit") { focusedField = nil }
```

- `@FocusState` — tracks focused control; bind via `.focused($focus, equals: value)` or `.focused($focus)` for Bool.
- `.focusable(_:)` — make any view focusable (macOS/iPadOS keyboard nav).
- `.focusSection()` / `.defaultFocus(_:priority:)` — focus order/priority in sections.
- `.focusedSceneValue` — share focused value up to scene level.
- `.onSubmit` — Return key handling; `.submitLabel(.next/.done/.search/.go)`.
- Keyboard avoidance: SwiftUI auto-adjusts scroll views for keyboard; `.scrollDismissesKeyboard(.interactively)`.
- tvOS/macOS: focus engine drives everything — test with keyboard only.

## 8. Drag & drop deep

```swift
// Draggable
NoteRow(note).draggable(note) { preview in
  // Custom preview
  Label(note.title, systemImage: "doc.text")
}
// UniformTypeIdentifiers inferred from Codable/Transferable

// Drop destination
List(notes) { note in ... }
  .dropDestination(for: Note.self) { items, location in
    addNotes(items); return true
  } isTargeted: { targeted in highlight = targeted }

// Delegate-based for custom types
.dropDestination(for: PhotoItem.self, delegate: DropDelegate())
```

- `Transferable` protocol — types that can be dragged: Codable+Transferable synthesis; `transferRepresentation` for custom.
- `UTType` — uniform type identifiers: `.plainText`, `.url`, `.image`, `.fileURL`, `.data`, `.json`, `.pdf`, custom `UTType(exportedAs:)`.
- `DropInfo` — `hasItemsConforming`, `itemProviders`, `location`, `localSession`.
- `DropProposal` — `dropIsPrecise`, `prefersFullSizePreview`.
- `.draggable(_:preview:)` — preview view builder; `.dragPreview` modifiers for shape/clipping.
- Spring loading: `.springLoadingDisabled()`.
- Cross-app drag: pasteboard-backed; within-app uses local session (faster, richer).

## 9. Representables — deep

```swift
struct CameraPreview: UIViewRepresentable {
  let session: AVCaptureSession

  func makeUIView(context: Context) -> PreviewView {
    let view = PreviewView()
    view.previewLayer.session = session
    view.previewLayer.videoGravity = .resizeAspectFill
    return view
  }
  func updateUIView(_ uiView: PreviewView, context: Context) { /* sync props */ }
  func makeCoordinator() -> Coordinator { Coordinator(self) }
  static func dismantleUIView(_ uiView: PreviewView, coordinator: Coordinator) {
    uiView.previewLayer.session = nil   // cleanup
  }

  class Coordinator { let parent: CameraPreview; init(_ p: CameraPreview) { parent = p } }
}
```

- Lifecycle: `makeUIView` once → `updateUIView` on state changes → `dismantleUIView` on removal (iOS 16+; use for cleanup — invalidate timers, remove observers).
- `sizeThatFits(_:uiView:context:proposedSize:)` — control sizing: return `UIView.sizeThatFits`/`systemLayoutSizeFitting` results; default = UIView's fitting.
- `UIViewControllerRepresentable` — same pattern for VCs; `updateUIViewController`, `dismantleUIViewController`.
- Coordinator: target for delegates/KVO/notifications — holds `parent` weakly? (struct parent is a value; store needed state).
- `NSViewRepresentable`/`NSViewControllerRepresentable` — macOS equivalents.
- `UIViewRepresentable.SizeThatFits` — return proposed vs fitting size; `.unspecified` handling.
- Avoid: recreating heavyweight UIKit views on every update — keep stable identity, mutate in update.
- Two-way data: bindings into UIKit via `updateUIView` + coordinator callbacks into SwiftUI state.

## 10. Environment & scene plumbing deep

- `EnvironmentKey` protocol (legacy): `static var defaultValue`; `EnvironmentValues` subscript extension.
- `@Entry` macro (iOS 18+): `@Entry var myValue: String = ""` — generates key+default automatically.
- Environment flows DOWN only; overrides via `.environment(\.key, value)` or `.environment(object)` for @Observable classes.
- Custom environment objects: `.environment(store)` + `@Environment(Store.self)`.
- `FocusedValue`/`FocusedObject` — share values with focused scene elements (commands access).
- Scene phases: `\.scenePhase` — `.active`/`.inactive`/`.background`/`.foregroundInactive`? (iOS: active/inactive/background).
- `openWindow(id:)` — open `Window("Title", id: "settings")`; `dismissWindow(id:)`.
- `handlesExternalEvents(matching:)` — macOS: route URLs/userActivities to existing window.
- `defaultLaunchBehavior(.suppressed)` — don't auto-open window on launch.
- `WindowResizability` — `.contentMinSize`, `.contentSize`, `.automatic`.
- Commands: `CommandGroup(replacing: .newItem)`, `CommandMenu("Tools")`, `.commandsRemoved`, `.commands { }` on views.
- `keyboardShortcut` phases: `.down`, `.up`, `.repeat`.

## 11. Transactions & animation deep

```swift
var transaction = Transaction()
transaction.animation = .spring(response: 0.4, dampingFraction: 0.8)
transaction.disablesAnimations = true
withTransaction(transaction) { state = newState }

// completion-based animation (iOS 17+)
withAnimation(.spring()) {
  state = newState
} completion: {
  // runs after animation completes
}

// completion criteria
withAnimation(.default, completionCriteria: .removed) { ... } completion: { }
// .logicallyComplete, .removed
```

- `Transaction` — carries animation + `disablesAnimations` + `isContinuous` + custom values down the tree; `.animation` modifier sets transaction.
- `withAnimation(_:completionCriteria:_:completion:)` — completion callbacks.
- `.animation(_, value:)` — animates when value CHANGES; the correct scoped form.
- `Animatable` protocol — custom animatable: `var animatableData: VectorArithmetic`; interpolate manually.
- `AnimatablePair`, `VectorArithmetic` — composite animation values.
- `phaseAnimator` — phases array + content closure `(content, phase)` + animation closure; for multi-stage animations.
- `keyframeAnimator` — `KeyframeTrack` values: `.linear`, `.easeIn`, `.easeOut`, `.easeInOut`, `.spring`, `.cubic`, `.move`, `.scale`, `.opacity`, `.rotation`, custom `Animatable` — trigger-based.
- `Animation` types: timing curves, springs (`.spring`, `.bouncy`, `.smooth`, `.snappy`, `.interactiveSpring`), `.linear`, `.easeInOut`, `.timingCurve(_:_:_:_:)`, `.interpolatingSpring(mass:stiffness:damping:initialVelocity:blendDuration:)`.
- `.transaction { $0.animation = nil }` — disable inherited animation for subtree.

## 12. Scroll APIs deep (iOS 17-27)

```swift
ScrollView {
  LazyVStack(spacing: 0) { ForEach(items) { Card($0).containerRelativeFrame(.vertical) { h, _ in h * 0.8 } } }
  .scrollTargetLayout()                    // children become snap targets
}
.scrollTargetBehavior(.viewAligned)        // snap to views; .paging for pages; .viewAlignedLimitBehavior
.scrollPosition(id: $scrollID, anchor: .top)  // programmatic; $scrollID: Item.ID?
.scrollIndicators(.hidden, axes: .vertical)
.scrollBounceBehavior(.basedOnSize, axes: .vertical)
.scrollClipDisabled()
.scrollDisabled(true)
.scrollDismissesKeyboard(.immediately / .interactively)
.defaultScrollAnchor(.top / .bottom / UnitPoint)
.scrollEdgeEffectStyle(.soft / .hard / .automatic)   // header/footer material
.onScrollGeometryChange(for: ScrollGeometry.self) { geo in geo } action: { old, new in ... }
.visualEffect { content, proxy in content.offset(y: proxy.frame(in: .scrollView).minY * 0.5) }
.scrollTransition { content, phase in content.opacity(phase.isIdentity ? 1 : 0) }
```

- `ScrollGeometry` — contentOffset, contentSize, containerSize, visibleRect, bounds.
- `scrollPosition(id:)` — two-way binding to topmost visible item ID (with `anchor:` for alignment).
- `scrollTargetLayout` + `scrollTargetBehavior(.viewAligned)` — carousel/paging without UIKit.
- `containerRelativeFrame` — size children relative to scroll container (hero cards).
- `onScrollGeometryChange` — observe offset/size for parallax, collapsing headers, progress.
- `visualEffect` — cheaper than GeometryReader for scroll-linked transforms (runs in render phase).
- `scrollTransition` — animate items as they scroll in/out of view.
- `safeAreaInset(edge:)` — bars pinned above scroll content (search, toolbar) that push content.
- `contentMargins` — padding inside scroll content.

## 13. Text deep

- `Text` + markdown: `Text("**bold** *italic* [link](url)")` — inline markdown subset (bold, italic, strikethrough, code, links); `AttributedString(markdown:)` for full parsing (headings, lists, images options).
- `AttributedString` attributes: `.font`, `.foregroundColor`, `.backgroundColor`, `.inlinePresentationIntent` (header levels, code block, blockquote, list items), `.link`, `.textHighlightColor`? — check names.
- `TextField`/`TextEditor` formatting: `FormatStyle` bindings — `.number`, `.percent`, `.date`, `.currency(code:)`, `.measurement`, `.byteCount`, `.duration`, `.relativeDate`, `.list`, `.verbatim`.
- `TextField` prompt + axis: `TextField("Note", text: $text, axis: .vertical)` — grows vertically (iOS 16+).
- `TextEditor` — plain text only (no attributed on iOS; macOS supports NSAttributedString via representable).
- `privacySensitive()` — mask in screen recording/notifications.
- `redacted(reason: .placeholder)` — skeleton UI; `.unredacted()`.
- `lineLimit(_:reservesSpace:)` — reserve space for N lines.
- `truncationMode`, `minimumScaleFactor`, `multilineTextAlignment`, `kerning`, `tracking`, `baselineOffset`.
- `TextRenderer` — custom text layout: `sizeThatFits`, `render`; draw glyphs manually.
- `typesettingLanguage` — hyphenation/language.

## 14. Charts (Swift Charts) deep

```swift
Chart(data) { item in
  BarMark(x: .value("Day", item.day), y: .value("Count", item.count))
    .foregroundStyle(by: .value("Type", item.type))
  RuleMark(y: .value("Avg", average))
  PointMark(x: .value("Day", item.day), y: .value("Count", item.count))
}
.chartXAxis { AxisMarks(values: .stride(by: 1)) { value in AxisGridLine(); AxisValueLabel() } }
.chartYAxis { AxisMarks(position: .leading) { value in AxisValueLabel { Text(value.as(Double.self)!, format: .number.precision(.fractionLength(0))) } } }
.chartLegend(position: .bottom)
.chartForegroundStyleScale(domain: ["A","B"], range: [.blue, .orange])
.chartScrollableAxes(.horizontal)
.chartXSelection(value: $selectedDay)
.chartPlotStyle { plotArea in plotArea.background(.quaternary.opacity(0.3)) }
.chartBackground { ChartBackgroundProxy { context in Text("\(context.plot.frame)") } }
.chartOverlay { proxy in /* custom marks via context.plot */ }
```

- Marks: `BarMark`, `LineMark`, `AreaMark`, `PointMark`, `RuleMark`, `RectangleMark`, `SectorMark` (pie/donut), `ArcMark`? — check, `BoxPlotMark`, `EllipseMark`, `RectangleMark`.
- Scales: `.value` encoding, `chartForegroundStyleScale`, `chartSymbolScale`? — `chartSymbol` for point styles.
- Axes: `AxisMarks`, `AxisGridLine`, `AxisTick`, `AxisValueLabel`, `AxisMarks(position:)`, preset values `.stride`, `.automatic`, `.stride(by:)`, custom `AxisMarkValues`.
- Selection: `chartXSelection`, `chartYSelection` bindings; `chartAngleSelection` for polar.
- Scrollable charts: `chartScrollableAxes` + `chartXVisibleDomain`.
- Annotations: `.annotation(position: .overlay/.top/.bottom) { Text }`.
- `ChartProxy` in overlay/background — `plot.frame(in:)`, `value(atX:)` conversions.
- Accessibility: charts get automatic summaries; `.accessibilityChartDescriptor` for custom.

## 15. Liquid Glass deep (OS 26/27)

```swift
.glassEffect()                                    // standard glass
.glassEffect(.regular.tint(.blue).interactive())   // tinted + interactive (macOS click response)
.glassEffectID("button", in: namespace)            // morph between glass elements
.glassEffectTransition(.matchedGeometry)          // transition style
.backgroundExtensionEffect()                      // extend background beyond bounds? (newer API)
.scrollEdgeEffectStyle(.soft)                     // scroll edge material
.tabViewBottomAccessory { ... }                   // accessory above tab bar
```

- `.glassEffect()` variants: `.regular`, `.clear`, `.identity`? — `Glass` struct: `.regular`, `.clear`, `.tint(Color)`, `.interactive(Bool)`, `.union(id:namespace:)`.
- `glassEffectID` + `glassEffectTransition(.matchedGeometry)` — morphing glass between states (like matchedGeometry for glass).
- Interactive glass responds to pointer on macOS; on iOS responds to touch.
- Toolbar/sidebar/tab bars auto-adopt Liquid Glass — don't override with opaque backgrounds.
- User tint slider affects glass appearance — test with different tint values.
- `backgroundExtensionEffect` — extend content background into safe areas? (verify exact API name in Xcode 27).
- `.presentationBackground` — sheet/popover backgrounds can be glass/materials.

## 16. Controls advanced

- `Button` roles: `.destructive` (red), `.cancel` — semantic styling.
- `Button` styles: `.bordered`, `.borderedProminent`, `.plain`, `.glass`, `.glassProminent` (iOS 26+), `.borderless`, `.link` (macOS).
- `Toggle` styles: `.switch`, `.checkbox`, `.button`.
- `Picker` styles: `.menu`, `.segmented`, `.wheel`, `.inline`, `.navigationLink`, `.palette`.
- `DatePicker` styles: `.compact`, `.graphical`, `.wheel`, `.stepperField`, `.automatic`.
- `Gauge` styles: `.linear`, `.circular`, `.accessoryCircular`.
- `LabeledContent` — form rows with label+content; `.labeledContentStyle(.titleAndIcon)`.
- `ControlGroup` — macOS grouped controls.
- `DisclosureGroup` — expandable content; `isExpanded` binding.
- `OutlineGroup`/`List` with `children:` keypath — disclosure trees.
- `ColorPicker`, `DatePicker`, `Stepper` (`onIncrement`/`onDecrement`), `Slider` (`in:step:`/`in:logs:`).
- `TextField` axis vertical, `lineLimit` for multiline.
- `SecureField` — same API, secure entry.
- `SearchBar` via `.searchable` — placement `.navigationBarDrawer`, `.sidebar`, `.toolbar`, `.automatic`.
- `Menu` — `primaryAction:` for split button; `menuStyle(.button/.borderlessButton/.borderlessButton)`.
- `ShareLink` — `item:`, `subject:`, `message:`, `preview:`.
- `Link` — `Link(destination:label:)`; `OpenURLAction` environment for custom handling: `.environment(\.openURL, OpenURLAction { url in ... })`.

## 17. Performance deep

- **Body evaluation cost**: every state change re-evaluates `body` of views that read that state. Split views so only affected subtrees re-eval; `EquatableView`/`.equatable()` skips when inputs equal (needs `==`).
- **Preference updates**: propagate up and invalidate ancestors — throttle/coalesce payload updates.
- **Layout invalidation**: `.frame`/`padding`/alignment changes are cheap; `GeometryReader` breaks layout caching — prefer `onGeometryChange`/`visualEffect`/`containerRelativeFrame`.
- **Lazy stacks**: `LazyVStack`/`LazyHStack`/`LazyVGrid` only render visible — required for long lists; eager stacks render all.
- **ForEach identity**: stable IDs prevent unnecessary row rebuilds; `id: \.persistentModelID` for SwiftData.
- **Image handling**: downsample before display (`ImageIO` thumbnail or `UIImage.preparingForDisplay`); AsyncImage caches but verify size; `.drawingGroup()` for complex static vector content.
- **Canvas**: use for custom 2-D; `symbols:` closure for reusable drawing; pair with `TimelineView` for animation — cheaper than many views.
- **Instruments SwiftUI template**: shows body eval count/time per view type, view properties, update reasons — THE tool for SwiftUI perf.
- **os_signpost** around suspected slow body evals.
- **@Observable granularity**: views reading only needed properties re-render less — split fat models.
- **`.task` vs `.onAppear`**: task is async + auto-cancels on disappear; onAppear is sync every-appear.
- **Avoid**: heavy work in `body`, `Date()`/`UUID()` in body (non-deterministic → infinite updates), `@State` objects created in body, GeometryReader in lists.

## 18. Testing SwiftUI views

- **ViewInspector** (nalexn): inspect view hierarchy in unit tests — `view.inspect()`, find views by type, read values, simulate taps.
- **Snapshot via ImageRenderer**: `ImageRenderer(content: view).uiImage` → compare bytes.
- **XCUITest**: `performAccessibilityAudit()`, accessibility identifiers for stable queries.
- **Preview-based**: `#Preview` + manual; for CI use snapshot libs or ViewInspector.
- **State testing**: test view models separately (@Observable classes) — keep logic out of views.

## 19. Common advanced pitfalls

| Pitfall | Fix |
|---|---|
| matchedGeometryEffect jumps | ensure only one `isSource: true` per id; stable namespace |
| Transition doesn't fire | transitions need insert/remove inside animation; use contentTransition for value changes |
| Preference loop | child writes pref that parent reads and writes back — guard with equality check |
| Layout protocol crash | subviews.count mismatch with placed indices — iterate `subviews.indices` |
| Focus stuck | `@FocusState` reset on view disappear; use `.defaultFocus` for initial |
| Drop destination not firing | ensure `UTType` matches draggable type; check `hasItemsConforming` |
| Representable update loop | updateUIView mutating state that triggers update — guard with equality |
| `@State` in representable recreated | representable is struct — store mutable state in Coordinator |
| Scroll position binding nil | `scrollPosition(id:)` needs items with stable IDs conforming to Hashable |
| Charts selection wrong | selection binding type must match mark value type exactly |
| Liquid Glass invisible | glass needs content behind it; check `.background`/material underneath |
