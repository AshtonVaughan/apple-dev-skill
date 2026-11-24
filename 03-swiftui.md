# SwiftUI Reference (2027 releases / Xcode 27 / OS 27)

The declarative UI framework for iOS, iPadOS, macOS, watchOS, tvOS, visionOS. Read `02-concurrency.md` for `@MainActor`/observation details.

## 1. Mental model — get this right first

- A `View` is a **value-type description** of UI, not a rendered object. `body` is a pure-ish function: given current state, produce a view tree description.
- The framework diffs view descriptions by **structural identity** (position in tree + explicit `.id()`). Two views at the same position with same type are "the same view" — their state persists.
- `body` re-evaluates when **observed state changes**: `@State`, `@Binding` source, `@Observable` tracked properties, `@Environment` values, `@Query` results.
- Views are cheap to create; expensive work belongs in models, not `body`. Never do I/O, heavy computation, or mutation in `body`.
- `body` must return quickly; the framework may call it speculatively. Side effects go in `.task`, `.onAppear`, button actions — never directly in `body`.
- "Why won't my view update?" → the state it reads isn't observed (plain `var` on non-observable class, `let`, value passed without binding, environment key not injected).

## 2. State & data flow — the wrapper table

| Wrapper | Owns? | Type | Use for |
|---|---|---|---|
| `@State` | Yes (view-owned) | value | view-local ephemeral state: text fields, toggles, animation flags, sheet presentation |
| `@Binding` | No | projected `$x` | two-way connection to parent's `@State`/`@Observable` prop |
| `@StateObject` | Yes (once) | class (ObservableObject) | view-owned reference model — legacy; prefer `@State` + `@Observable` class |
| `@ObservedObject` | No | class | external ObservableObject passed in |
| `@EnvironmentObject` | No | class | shared ObservableObject via `.environmentObject()` — legacy; prefer `@Environment(MyStore.self)` with `@Observable` |
| `@Environment` | No | keyed lookup | system values (colorScheme, dismiss, openWindow, scenePhase, layoutDirection, dynamicTypeSize, pixelLength) + custom `EnvironmentKey`/`@Entry` |
| `@AppStorage` | Yes | UserDefaults-backed | small persisted settings (Bool, String, Int, Double, URL, Data) |
| `@SceneStorage` | Yes | scene-scoped | state preserved per scene (e.g. navigation path per window) |
| `@FocusedState` | Yes | Hashable? | focus tracking for controls (`.focused($isFocused)`) |
| `@Namespace` | n/a | ID generator | `matchedGeometryEffect`, scroll targeting |
| `@Query` | Yes | SwiftData | Core Data/SwiftData fetch results in view |
| `@Bindable` | No | `@Observable` class | create bindings into an observable model's properties (`$model.name`) |
| `@Observable` (macro on class) | — | class | THE modern model: fine-grained observation, no Combine needed |

Rules:
- `@State` for value types owned by the view; initialize in `init` for computed defaults: `@State private var count = 0` or `_count = State(initialValue:)` for params.
- `@Observable` classes: mark properties `var` (tracked) vs `let` (untracked). Methods that mutate can be plain. `private(set)` works.
- Environment DI pattern: `.environment(store)` injects; `@Environment(Store.self)` reads. For optionals use `@Environment(Store.self) private var store: Store?` — SwiftUI provides optional environment lookup.
- `@Entry` macro (iOS 18+) declares custom environment keys with less boilerplate than `EnvironmentKey`.

## 3. Layout system

### Stacks & grids
```swift
VStack(alignment: .leading, spacing: 12) { ... }   // vertical
HStack { ... }                                       // horizontal
ZStack { ... }                                       // layered
LazyVStack / LazyHStack                              // lazy = only visible rendered (in ScrollView)
Grid { GridRow { ... } }                             // 2-D grid, iOS 16+
LazyVGrid(columns: [GridItem(.flexible()), GridItem(.fixed(80))]) { ... }
Table(of:columns:rows:selection:)                    // real table, macOS/iPadOS 16+
List { Section("Header") { ForEach(items) { row } } } // list w/ sections, styles
```
- Spacing params: `spacing:` fixed or nil = platform default; `HStack(spacing: 0)` tight.
- Alignment: `.center`, `.leading`, `.topTrailing`, `.firstTextBaseline` etc. — axis-relative.
- `frame(minWidth:idealWidth:maxWidth:minHeight:idealHeight:maxHeight:alignment:)` — the workhorse. `.frame(maxWidth: .infinity)` to expand.
- `fixedSize()` — view takes ideal size, ignoring proposed size (use sparingly — can break scroll).
- `layoutPriority(_:)` — higher priority views get space first when space is tight.

### Geometry & sizing
- **Safe area**: content avoids notch/home indicator by default; `.ignoresSafeArea(edges:)` to extend; `.safeAreaInset(edge:)` for bars that push content (search bars, toolbars).
- `GeometryReader` — gives size via preference, BUT it expands to fill and breaks layout caching; prefer `onGeometryChange(for:of:action:)` (iOS 16+) or `visualEffect`/`containerRelativeFrame` for size-relative effects.
- `containerRelativeFrame(_:_:)` — size relative to nearest container (iOS 17+).
- `ViewThatFits` — picks first child that fits proposed size.
- Custom layouts: `Layout` protocol — `sizeThatFits(proposal:subviews:cache:)` + `placeSubviews(in:proposal:subviews:cache:)`; `Cache` for expensive subview measurement.
- `AlignmentGuide` — custom alignment lines for stacks.

### Scroll
```swift
ScrollView(.vertical) { LazyVStack { ForEach(...) { Row($0) } } }
.scrollPosition(id: $scrolledID)                    // programmatic position (iOS 17+)
.scrollTargetLayout()                               // marks children as snap targets
.scrollTargetBehavior(.paging / .viewAligned / .paging)  // snap scrolling
.scrollBounceBehavior(.basedOnSize)                 // disable bounce when content fits
.scrollIndicators(.hidden)
.scrollClipDisabled()                               // content can overflow bounds
.onScrollGeometryChange(for:of:action:)             // scroll offset/size observation (iOS 18+)
```
- `ScrollViewReader` + `proxy.scrollTo(id, anchor:)` for imperative scroll (older API, still works).
- `scrollTransition` / `scrollTargetBehavior` for carousel effects; `visualEffect` for scroll-linked transforms.

## 4. Controls — the catalog

```swift
Button("Title") { action }                          // or Button(action:label:)
  .buttonStyle(.borderedProminent / .bordered / .plain / .glassProminent?)
  .controlSize(.small/.regular/.large)
  .keyboardShortcut("s", modifiers: .command)       // macOS/iPadOS hw kb
Toggle("Wi-Fi", isOn: $on)                          // .switch/.checkbox/.button styles
Picker("Sort", selection: $sel) { ForEach(...){ Text($0).tag($0) } }  // .menu/.wheel/.segmented/.radioGroup
Slider(value: $v, in: 0...1) { editing in }         // onEditingChanged
DatePicker("Date", selection: $date, displayedComponents: .date)
TextField("Placeholder", text: $text)               // .textFieldStyle(.roundedBorder)
SecureField("Password", text: $pw)
TextEditor(text: $body)                             // multi-line rich-ish editing
ProgressView()                                      // determinate: ProgressView(value:total:)
Gauge(value:in:currentValueLabel:)                  // iOS 16+
Label("Settings", systemImage: "gear")               // icon+text
Link("Website", destination: url)                   // opens URL
ShareLink(item: url)                                // share sheet button
```
- `.searchable(text:placement:prompt:)` — search UI, `.searchSuggestions`, `.searchScopes`.
- `.disabled(_:)`, `.opacity`, `.allowsHitTesting(_:)` (vs `.disabled` — hit-testing still matters for gestures), `.contentShape(Rectangle())` for expanding tap area.
- `.contextMenu { } preview: { }` — long-press menu with preview.
- `.swipeActions(edge:allowsFullSwipe:)` on List rows (iOS 15+); **`.swipeActionsContainer` on any ScrollView** — new 2027 API, enables swipe actions outside List.
- `.confirmationDialog` / `.alert` — **2027: item-binding pattern** like sheets: `.alert(item: $selected)` style APIs appear automatically when bound value set.
- `Menu { } label: { }` + `primaryAction:` for split-button behavior.
- `ControlGroup` — grouped controls on macOS; `LabeledContent` for form rows; `Form` style `.grouped`/`.automatic`.
- Keyboard: `.submitLabel(.next/.done/.search)`, `.onSubmit`, `.focused`, `.focusedSceneValue`, `.focusSection` (tvOS/macOS), `.defaultFocus`.
- `.keyboardShortcut` — modifiers `.command`, `.shift`, `.option`, `.control`, `.capsLock`; `.keyboardShortcut(.defaultAction)` = Return.

## 5. Text

```swift
Text("Hello")                    // LocalizedStringKey — looks up localization
Text(verbatim: "Raw")            // no localization
Text("Score: \(score)")          // interpolated LocalizedStringKey
Text(AttributedString)           // styled: .font/.foregroundColor/.bold() etc.
Text("**Bold** and *italic*")    // markdown inline (iOS 15+); .markdownTextStyle? no — use AttributedString(markdown:)
TextField("Name", value: $name, format: .number)   // FormatStyle binding (iOS 15+)
```
- `Font`: `.title/.headline/.body/.caption/.system(size:weight:design:)`, `.custom("Name", size:)`, `Font.TextStyle` scaling with Dynamic Type.
- `TextRenderer` protocol — custom text drawing/layout (iOS 16+).
- `.textSelection(.enabled)` — selectable text (iOS 15+).
- `.lineLimit`, `.truncationMode`, `.minimumScaleFactor`, `.multilineTextAlignment`, `.lineSpacing`.
- `TextEditor` supports attributed text via `NSAttributedString` bridging on macOS; plain on iOS.

## 6. Images & media

```swift
Image(systemName: "gear")                        // SF Symbol — config: .symbolRenderingMode(.hierarchical/.palette/.multicolor), .symbolVariant(.fill), .imageScale, .foregroundStyle
Image("assetName")                               // asset catalog
Image(decorative: uiImage)                       // UIImage/NSImage
AsyncImage(url: url) { phase in                  // async loading
  switch phase { case .success(let img): img.resizable(); case .failure: Image(systemName:"exclamationmark.triangle"); default: ProgressView() }
}                                                 // 2027: standard HTTP caching by default
VideoPlayer(player: avPlayer)                     // AVKit
Map(initialPosition: .region(...)) { Marker(...) } // MapKit SwiftUI (iOS 17+)
```
- SF Symbols: 6,000+ symbols, hierarchical rendering, variable draw (with `.contentTransition(.symbolEffect)`), animation via `symbolEffect(.bounce/.pulse/.variableColor)` + `.symbolEffect` APIs.
- `.resizable().scaledToFit()/.scaledToFill()` + `.clipped()` — standard image sizing chain.
- SF Symbol naming: `square.and.arrow.up`, `chevron.right`, `xmark`, `magnifyingglass`, `person.crop.circle` — browse in SF Symbols app.
- Canvas: `Canvas { context, size in } symbols: { }` — 2-D drawing with `GraphicsContext` (fill/stroke/clip/drawLayer/shadow/blend modes); pair with `TimelineView` for animation.
- TimelineView: `TimelineView(.animation/.periodic/.everyMinute)` — redraw on schedule.
- `.drawingGroup()` — Metal-render layer contents (faster for complex static content); `.compositingGroup()` — flatten alpha.

## 7. Navigation

### Stack (push/pop)
```swift
NavigationStack(path: $path) {           // path: [Route] or NavigationPath — value-based
  List(items) { NavigationLink(value: $0) { Text($0.name) } }
  .navigationDestination(for: Item.self) { item in DetailView(item) }
}                                        // deep links: path.append(route)
```
- Value-based navigation preferred over `NavigationLink(destination:)` (string/type-erased older API).
- `.navigationTitle()`, `.navigationBarTitleDisplayMode(.inline/.large)`, `.toolbar { ToolbarItem(placement:) { } }`.
- Toolbar placements: `.primaryAction`, `.confirmationAction`, `.cancellationAction`, `.navigation`, `.principal`, `.secondaryAction`, `.topBarLeading/Trailing`, `.bottomBar`, `.keyboard`, `.status`, `.sidebar`, `.header`/`.footer` (macOS/iPadOS).
- **2027 toolbar APIs**: `.toolbarVisibilityPriority`, `.toolbarOverflowMenu`, `.topBarPinnedTrailing`, `.toolbarMinimizeBehavior` — control collapse/overflow on resize.

### Split (iPad/macOS)
```swift
NavigationSplitView { Sidebar() } content: { ContentList() } detail: { Detail() }
// columnVisibility: .automatic/.doubleColumn/.all/.detailOnly
// .navigationSplitViewColumnWidth(min:ideal:max:)
```

### Modal presentation
```swift
.sheet(isPresented: $show) { SheetView() }                 // detents: .medium/.large, .presentationDetents([.height(200)])
.fullScreenCover(isPresented: $show) { ... }
.popover(isPresented: $show, arrowEdge: .top) { ... }      // iPad only
.alert("Title", isPresented: $show) { Button("OK"){} } message: { Text("...") }
.confirmationDialog("Delete?", isPresented: $show) { ... }
// 2027: item-binding — .alert(item: $item), .confirmationDialog(item:) auto-present when set
.inspector(isPresented: $show) { InspectorView() }         // trailing panel (iPad/macOS)
```
- `.presentationDetents([.medium, .large])`, `.presentationDragIndicator(.visible)`, `.interactiveDismissDisabled()`.
- Sheets get their own environment; use `@Environment(\.dismiss)` for close.
- `fullScreenCover` for immersive flows; `sheet` for quick tasks.

### Tabs
```swift
TabView { Tab("Home", systemImage: "house") { HomeView() } Tab("Settings", systemImage:"gear") { SettingsView() } }
// or TabView(selection: $tab) + .tabItem (older); .tabViewStyle(.sidebarAdaptable) on iPad/macOS
```

## 8. Lists, tables, forms

```swift
List(items, selection: $sel) { item in Row(item).tag(item) }   // selection binding
  .listStyle(.insetGrouped / .sidebar / .plain / .grouped)
  .listSectionSeparator(.hidden)
  .refreshable { await reload() }                              // pull-to-refresh
  .swipeActions(edge: .trailing) { Button(role: .destructive) {} }  // + .tint, .labelStyle
  .onMove { from, to in } / .onDelete { indexSet in }          // edit mode
  .environment(\.editMode, .constant(.active))

Table(of: Item.self, columns: {                                        // macOS/iPadOS
  TableColumn("Name", value: \.name)
  TableColumn("Value") { item in Text(item.formatted) }
}, rows: { items })

OutlineGroup(items, children: \.children) { item in Row(item) }       // disclosure tree
Form { Section { Toggle...; Picker... } } .formStyle(.grouped)
```
- `ForEach(items, id: \.self)` or items must be `Identifiable`; stable IDs prevent state loss on reorder.
- `.listRowSeparator(.hidden)`, `.listRowBackground`, `.listRowInsets`, `.alignmentGuide` for custom row layout.
- `LazyVStack` inside `ScrollView` for custom grids; `List` for standard row behavior + swipe/actions/edit.
- 2027: **reorderable containers** — drag-to-reorder in List/LazyVGrid/any container via `.draggable`/`.dropDestination` unified APIs and `reorderable` container APIs (same code across containers, watchOS too).

## 9. Animation & transitions

```swift
withAnimation(.spring(response: 0.3, dampingFraction: 0.7)) { state.toggle() }
.animation(.easeInOut(duration: 0.2), value: state)   // implicit on value change
.transition(.opacity.combined(with: .move(edge: .bottom)))
.matchedGeometryEffect(id: "hero", in: namespace)      // shared-element transitions
.symbolEffect(.bounce, value: count)
.phaseAnimator([false, true]) { content, phase in ... } animation: { ... }
.keyframeAnimator(initialValue: ..., trigger: ...) { ... }
```
- `Animation`: `.default`, `.easeInOut`, `.spring()`, `.bouncy`, `.smooth`, `.snappy`, `.linear`, `.interactiveSpring`, `.timingCurve`, `.interpolatingSpring(mass:stiffness:damping:initialVelocity:)`.
- `Transaction`: `.animation` inside `withTransaction`; `transaction { $0.animation = ... }` on views.
- Transitions apply on insert/remove in containers (`if/else` in stacks, `ForEach` changes).
- `matchedGeometryEffect` — THE hero-transition API; requires `@Namespace`.
- `.animation(_:value:)` preferred over blanket `.animation(_:)` (deprecated pattern — animates on any change).
- `Animatable` protocol for custom animatable properties; `animatableData`.
- Scroll-linked: `.scrollTransition(.interactive)` on scroll children; `visualEffect` for per-frame effects.

## 10. Gestures

```swift
.gesture(DragGesture(minimumDistance: 10).onChanged { v in }.onEnded { v in })
.gesture(TapGesture(count: 2).onEnded { })           // double-tap
.simultaneousGesture(MagnifyGesture().onChanged { }) // with scroll/other
.highPriorityGesture(LongPressGesture(minimumDuration: 0.5).onEnded { ... })
.gesture(SpatialTapGesture(coordinateSpace: .global) { loc in })  // location-aware tap
.updating($isPressed) { value, state, _ in state = value.translation }  // gesture state during touch
```
- Composing: `.gesture(A.exclusively(before: B))`, `.gesture(A.simultaneously(with: B))`.
- `.gestureMask` — which gestures take precedence.
- `.onTapGesture`, `.onLongPressGesture` shortcuts.
- Custom `Gesture` protocol: `Value` type + `map`/`sequenced`/`exclusively` combinators.
- Drag & drop: `.draggable(item)`, `.dropDestination(for:delegate:)` — see `15-recipes.md`.

## 11. Sheets, popovers, alerts — modern patterns

```swift
// item-driven presentation (preferred — no separate Bool)
@State private var selectedItem: Item?
.sheet(item: $selectedItem) { item in DetailView(item: item) }

// isPresented + onDismiss for cleanup
.sheet(isPresented: $show, onDismiss: cleanup) { ... }

// multiple detents
.presentationDetents([.fraction(0.4), .medium, .large])
.presentationDragIndicator(.visible)
.presentationBackground(.thinMaterial)   // or Color/ShapeStyle
.presentationCornerRadius(24)
```

## 12. Environment & scene plumbing

```swift
@main struct MyApp: App {
  @State private var store = AppStore()
  var body: some Scene {
    WindowGroup { ContentView().environment(store) }
    Settings { SettingsView() }                    // macOS Settings scene (Cmd+,)
    MenuBarExtra("Widget", systemImage: "star") { MenuContent() }  // menu bar app
  }
}
```
- Common environment values: `\.colorScheme`, `\.colorSchemeContrast`, `\.layoutDirection`, `\.dynamicTypeSize`, `\.legibilityWeight`, `\.scenePhase`, `\.dismiss`, `\.openWindow`, `\.openImmersiveSpace`, `\.horizontalSizeClass`, `\.verticalSizeClass`, `\.displayScale`, `\.pixelLength`, `\.accessibilityReduceMotion`, `\.accessibilityVoiceOverEnabled`, `\.accessibilityReduceTransparency`, `\.accessibilityDifferentiateWithoutColor`, `\.legibilityWeight`, `\.controlActiveState`, `\.isPresented`.
- `@Environment(\.scenePhase)` — `.active`/`.inactive`/`.background` (iOS); use for pause/resume logic.
- `@Environment(\.dismiss)` — programmatic dismiss of sheets/fullScreenCover.
- `@Environment(\.openWindow)` — open named windows by ID (`Window("id", id: "main")`).
- `WindowGroup` vs `Window` vs `WindowResizability`: `.contentResizability(.contentSize)` etc.
- `commands { CommandGroup... }` — macOS menu commands; `.commandsRemoved`, `CommandMenu`.
- `defaultSize`, `defaultPosition`, `windowResizability`, `windowToolbarStyle`, `windowLevel` on scenes.
- `handlesExternalEvents(matching:)` — macOS: continue user activity in existing window.

## 13. Previews

```swift
#Preview("Light") { ContentView().environment(Store.mock) }
#Preview(traits: .sizeThatFitsLayout) { ... }
// Preview traits: .fixedLayout, .sizeThatFitsLayout, .landscapeLeft/.portrait, .dark/.light,
//   .dynamicType(...), .environment(\.colorScheme, .dark), .navigationSplitViewLayout(.balanced)
```
- `#Preview` macro replaces `PreviewProvider`; multiple previews per file; preview canvas interactive (iOS 17+ state preserved).
- **2027/Xcode 27**: Live Previews have **resize handles** — test iPhone-app-on-iPad/Mac resizing interactively.
- Preview snapshots for CI: `xcodebuild test -scheme ... -only-testing:PreviewTests` or snapshot libs; `ImageRenderer` renders a view to UIImage/NSImage for programmatic snapshots.
- Keep previews fast: mock models, small data, avoid network in preview (use `#Preview` + in-memory container or `ModelConfiguration(isStoredInMemoryOnly: true)`).

## 14. Liquid Glass & visual design (OS 26/27)

- `.glassEffect()` / `.glassEffectID(_:in:)` / `.glassEffectTransition` — Liquid Glass materials API (iOS 26+).
- `.background(.ultraThinMaterial/.thinMaterial/.regularMaterial/.thickMaterial/.ultraThickMaterial)` — frosted materials.
- Toolbar/sidebar automatically adopt Liquid Glass on OS 27 — no code needed; use `.interactive()` glass on macOS for click response.
- `.scrollEdgeEffectStyle` — header/footer blur under scroll.
- `.tabViewBottomAccessory` — accessory above tab bar (iOS 26+).
- Tint slider: users can adjust Liquid Glass tint — don't hardcode glass colors.
- Inactive window dimming on iPad/Mac is automatic.
- Design for resizability: avoid fixed frames; test Stage Manager, split view, iPhone mirroring (iOS 27 iPhone apps are resizable).

## 15. Accessibility

```swift
.accessibilityLabel("Add item")                    // VoiceOver label
.accessibilityValue("\(count) items")
.accessibilityHint("Double tap to add")
.accessibilityAddTraits(.isButton) / .accessibilityRemoveTraits
.accessibilityHidden(true)
.accessibilityElement(children: .combine)          // group children into one element
.accessibilitySortPriority(1)
.accessibilityRepresentation { Text("Custom") }    // replace element for AT
.accessibilityInputLabels(["add", "new"])         // Voice Control labels
.accessibilityActions { Button("Rename") {...} }   // custom rotor actions
```
- Dynamic Type: use text styles (`.font(.body)` etc.), avoid fixed `.font(.system(size:))` for body text; test with `.dynamicTypeSize(.accessibility3)` previews.
- `accessibilityReduceMotion` — gate nonessential animation.
- `accessibilityDifferentiateWithoutColor` — don't convey meaning by color alone.
- Contrast: `.foregroundStyle(.primary/.secondary)`, materials adapt automatically.
- Audit: XCUITest `performAccessibilityAudit()` (iOS 17+) catches issues automatically.
- Focus order: `.accessibilityFocused`? — use `.focusable()`/`defaultFocus` for keyboard/TV; macOS full keyboard access via `.keyboardShortcut` on everything.

## 16. Localization

- String Catalogs (`.xcstrings`) — Xcode 15+ standard; keys = source strings, values per locale, plural variations, automatic extraction from `Text("...")`/`String(localized:)`.
- `Text("key")` uses `LocalizedStringKey` — automatic lookup + interpolation of localized format strings.
- `String(localized: "key", defaultValue: "...", comment: "...")` — explicit; `String(localized:)` for non-View code.
- Plurals: `Text("^[\(count) item](inflect: true)")` or xcstrings plural rules; `String(localized:)` with `count:` param + `.stringsdict` legacy.
- Right-to-left: `.environment(\.layoutDirection, .rightToLeft)`; use leading/trailing not left/right (`.padding(.leading)` not `.padding(.left)`); `flipsForRightToLeftLayoutDirection` default.
- Dates/numbers: `Text(date, style: .date/.time/.relative)`, `.formatted()`, `FormatStyle` — locale-aware automatically.
- Xcode: Product > Export/Import Localizations; pseudolocalization scheme for testing.

## 17. Interop with UIKit/AppKit

```swift
struct WebView: UIViewRepresentable {              // iOS
  func makeUIView(context: Context) -> WKWebView { WKWebView() }
  func updateUIView(_ uiView: WKWebView, context: Context) { uiView.load(...) }
  func makeCoordinator() -> Coordinator { ... }     // delegate target
}
struct MacWebView: NSViewRepresentable { ... }       // macOS
UIViewControllerRepresentable — wrap UIViewController (e.g. PHPickerViewController)
```
- Coordinator pattern for delegates/data sources.
- `UIViewRepresentable.SizeThatFits` — control sizing negotiation.
- Reverse: `UIHostingController(rootView:)` embeds SwiftUI in UIKit; `NSHostingController`/`NSHostingView` on macOS; `UIHostingConfiguration` for list cells.
- SwiftUI Introspect (third-party) reaches into UIKit internals — avoid in production.

## 18. Performance rules

- `body` cost: keep it O(view count); extract subviews as separate `View` structs (gets own identity + diffing).
- `EquatableView` / `.equatable()` — skip re-render when inputs equal (explicit ==).
- Lazy stacks for long lists — `LazyVStack`/`LazyHGrid`; never `VStack`+`ForEach` for 1000+ rows.
- `ImageRenderer` off-main for snapshot rendering.
- Avoid `GeometryReader` in lists (breaks laziness); prefer `onGeometryChange`.
- `.drawingGroup()` for complex vector content; `.compositingGroup()` for blended groups.
- Profile: Instruments **SwiftUI template** (body eval count/time, view properties), **Animation Hitches**, **Time Profiler**, **Allocations**.
- `os_signpost` around suspected slow view updates; `.task` for async loads (auto-cancel on disappear).
- State granularity: split `@Observable` models so views read only what they need — observation is per-property.
- `@Query` in lists: fetch only visible sections; avoid fetching whole DB into view.

## 19. New in OS 27 / Xcode 27 (2026 releases)

- **Document API**: `WritableDocument`/`ReadableDocument` protocols — async incremental disk I/O, progress via Foundation `Subprogress`; `DocumentCreationSource` + `NewDocumentButton` for multiple creation sources. Replaces ad-hoc `fileDocument`/`referenceFileDocument` patterns for new apps.
- **Toolbar**: `.toolbarVisibilityPriority`, `.toolbarOverflowMenu`, `.topBarPinnedTrailing`, `.toolbarMinimizeBehavior` — fine control over collapse/overflow as windows resize.
- **Reorderable containers**: unified drag-reorder APIs for List/LazyVGrid/any container; first time on watchOS.
- **`.swipeActionsContainer`** on any ScrollView — swipe actions outside List.
- **Item-bound alerts/dialogs**: `.alert(item:)`/`.confirmationDialog(item:)` like sheets.
- **AsyncImage**: standard HTTP caching by default (respects server cache headers).
- **iOS 27**: iPhone apps resizable (iPhone Mirroring, iPad); Liquid Glass refinements + user tint slider; inactive-window dimming; Xcode 27 Live Preview resize handles.
- **Xcode 27 agent skills**: built-in SwiftUI skills teaching new-API adoption — use when generating SwiftUI.
- Foundation Models framework (iOS 26+, see `14-platforms-versions.md`): `LanguageModelSession`, `@Generable` guided generation — integrate via `AppIntent`/tools for Siri AI.
- App Intents are now central for Siri AI discovery — see `12-lifecycle-extensions.md`.

## 20. Common SwiftUI bugs → fixes

| Bug | Cause | Fix |
|---|---|---|
| View doesn't update | state not observed (plain class var, `let`, missing `@State`) | mark `@Observable`/`@State`/`@Published` |
| "Modifying state during view update" | mutating `@State` inside `body` or during another update | move to `.task`/`.onAppear`/action; use `DispatchQueue.main.async` only as last resort |
| Infinite layout loop | `GeometryReader` feedback, `.frame` fighting, negative padding | remove GeometryReader; check layout priorities; `.fixedSize` where needed |
| List rows jump/reorder wrong | unstable `id` (index-based, UUID regenerated) | stable `Identifiable` IDs from data |
| Sheet doesn't show | `isPresented` set in same update cycle as presenting view appears | set in `.task`/`.onAppear` after appear, or use item binding |
| NavigationLink does nothing | inside `List` without selection binding or wrong stack | use `NavigationLink(value:)` + `navigationDestination` |
| Animation not running | state change not wrapped / `.animation` on wrong value | `withAnimation` or `.animation(_, value:)` |
| Keyboard covers input | no scroll adjustment | `ScrollView` + `.scrollDismissesKeyboard`, safe area handles automatically |
| Dark mode wrong colors | hardcoded `.black`/`.white` | `.primary`/`.secondary`/semantic colors, asset catalog colors |
| Memory grows in list | non-lazy stack, images not downsampled | `LazyVStack`, downsample images before display |
| `@State` resets | view identity change (conditional `if` swaps type/position) | stable structure or lift state up |
| Preview broken | `@StateObject` in preview without injection | `#Preview { View().environment(Model()) }` |

## 21. Decision cheat sheet

- New app UI → SwiftUI everywhere; UIKit only for missing APIs.
- Model → `@Observable final class`; view-local → `@State`; child editing → `@Bindable`/`@Binding`.
- Persistence of settings → `@AppStorage`; scene state → `@SceneStorage`; real data → SwiftData (see `06-data-persistence.md`).
- Long list → `List` (rows+swipe+edit) or `LazyVStack` in `ScrollView` (custom layout); never eager stack.
- Master-detail → `NavigationSplitView`; phone flow → `NavigationStack`; quick task → `.sheet`; immersive → `fullScreenCover`.
- Custom 2-D drawing → `Canvas`; animated → `TimelineView`+Canvas or `Canvas` with `symbols`; complex → Metal.
- Need UIKit component → `UIViewRepresentable`; need SwiftUI in UIKit → `UIHostingController`.
