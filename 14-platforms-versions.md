# Platforms, Versions & Ecosystem

OS/Swift/Xcode version matrix, Catalyst vs native, per-platform notes, Apple Intelligence frameworks.

## 1. Version matrix (as of Nov 2026)

| Platform | Current | Released | Notes |
|---|---|---|---|
| **iOS** | 27 | Sept 14, 2026 | Siri AI, Liquid Glass refinements, resizable iPhone apps |
| **iPadOS** | 27 | Sept 14, 2026 | Stage Manager, pointer, external display |
| **macOS** | 27 "Golden Gate" | Sept 14, 2026 | Last macOS with Rosetta 2; Siri AI; Liquid Glass refinements |
| **watchOS** | 27 | Sept 14, 2026 | Smart Stack, complications, focus modes |
| **tvOS** | 27 | Sept 14, 2026 | Focus engine, top shelf, AVKit |
| **visionOS** | 27 | Sept 14, 2026 | Windows, volumes, spaces, immersion styles |
| **Xcode** | 27.2 | Nov 2026 | SwiftUI 2027 APIs, agent skills, Live Preview resize |
| **Swift** | 6.4.2 | Sept 15, 2026 | Swift Build, Subprocess 1.0, Span↔std::span, noncopyable arrays |

### Apple Intelligence availability (iOS 27)
- **Siri AI** — LLM-based, context-aware; on-device AFM 3 Core (3B params), AFM 3 Core Advanced (20B sparse, 1-4B active), AFM 3 Cloud/ADM 3 Cloud/AFM3 Cloud Pro (remote).
- **Languages**: English at launch; French, Japanese, Korean, Portuguese, Spanish added Oct 2026.
- **Regions**: NOT available in EU or China at launch (regulatory).
- **Devices**: iPhone 15 Pro+ and newer for full Siri AI; older devices get limited Apple Intelligence.

### macOS 27 specifics
- **Last macOS with Rosetta 2** — Intel apps still run; macOS 28 will drop it.
- **macOS 26 Tahoe** — last macOS for Intel Macs (security updates 3 more years).
- **Liquid Glass refinements**: materials adapt, `.interactive()` glass, inactive window dimming, user tint slider.
- **SwiftUI-first APIs**: most new features arrive SwiftUI-first.

## 2. Swift version history (key milestones)

| Version | Year | Key features |
|---|---|---|
| Swift 5.0 | 2019 | ABI stability, result builders preview |
| Swift 5.1 | 2019 | Property wrappers, opaque result types (`some`) |
| Swift 5.2 | 2020 | Multiple trailing closures, key path member lookup |
| Swift 5.3 | 2020 | `if let` shorthand? (no — 5.7), property wrappers on properties |
| Swift 5.4 | 2021 | Result builders, implicit returns, async/await preview |
| Swift 5.5 | 2021 | **Concurrency**: async/await, actors, Sendable; `if let` shorthand; `guard let` shorthand |
| Swift 5.6 | 2021 | `if/switch` expressions? (no — 5.9), typed throws preview, package access |
| Swift 5.7 | 2022 | `if let`/`guard let` shorthand, `some`/`any` in parameters, primary associated types, regex literals, `Sendable` improvements |
| Swift 5.8 | 2023 | `if let` shorthand? (already 5.7), `#Predicate` macro, `withObservationTracking` preview |
| Swift 5.9 | 2023 | **Macros** (freestanding/attached), `if`/`switch` expressions, parameter packs, `borrow`/`consuming` parameters, `~Copyable`, `~Escapable`, `package` access, `RegexBuilder` improvements |
| Swift 5.10 | 2024 | `Span`/`InlineArray`, `AsyncIteratorProtocol` improvements, `sending` parameters preview |
| Swift 6.0 | 2024 | **Strict concurrency**, typed throws, `any` required for existentials, region-based isolation, isolated conformances, `Mutex`/`Atomic` preview, `borrowing`/`consuming` official |
| Swift 6.1 | 2025 | `defaultIsolation` preview, `~Sendable` preview, `Span`/`InlineArray` stable, `Iterable` protocol, `AsyncSequence` improvements |
| Swift 6.2 | 2025 | `defaultIsolation(MainActor.self)`, `Synchronization` framework (Mutex/Atomic), `~Sendable` official, `withTaskCancellationShield` preview |
| Swift 6.3 | 2026 | Minor improvements, `Span` bridging improvements |
| Swift 6.4 | Sept 2026 | **Swift Build** default, **Subprocess 1.0**, `SE-0521` (`some P?`), `SE-0522` (`@diagnose`), `SE-0493` (async defer), `SE-0503` (`~Copyable`/`~Escapable` assoc types), `SE-0518` (`~Sendable`), `SE-0508` (trailing closures after literals), typed-throws Task init warnings, `withTaskCancellationShield`, noncopyable arrays, `Iterable` protocol, `Span`↔`std::span`, Java interop async, Wasm/JSKit 40×, Embedded Swift existential+errors |

## 3. SwiftUI feature timeline

| OS | Key SwiftUI features |
|---|---|
| iOS 13 | Debut, basic views |
| iOS 14 | `LazyVGrid`, `List` improvements, `App` lifecycle, `TabView` sidebar, `Menu`, `ProgressView` |
| iOS 15 | `.refreshable`, `AsyncImage`, `.searchable`, `NavigationStack`? (no — iOS 16), `TextField` format, `photosPicker`, `controlGroupStyle` |
| iOS 16 | `NavigationStack`, `NavigationSplitView`, `Table`, `Grid`, `Layout` protocol, `Canvas`, `PhotosPicker`, `ShareLink`, `onSubmit`, `presentationDetents`, `scrollDismissesKeyboard`, `accessibilityRepresentation` |
| iOS 17 | `Observable` framework, `@Observable`, `@Query` improvements, `scrollPosition`, `scrollTargetBehavior`, `contentTransition`, `symbolEffect`, `phaseAnimator`, `keyframeAnimator`, `containerRelativeFrame`, `onScrollGeometryChange`, `presentationBackground`, `presentationCornerRadius` |
| iOS 18 | `@Entry` macro, `tabViewBottomAccessory`, `ControlWidget`, `MeshGradient`, `scrollEdgeEffectStyle`, `buttonBorderShape`, `symbolRenderingMode` improvements, `ImageRenderer` improvements, `UIViewRepresentable` `sizingOptions` |
| iOS 26 | **Liquid Glass** design, `.glassEffect`, `tabViewBottomAccessory`, `scrollEdgeEffectStyle`, `UIGlassEffect`, `SearchTab`, `UIUpdateLink`, `SpeechAnalyzer`, `Translation` framework, `FoundationModels`, `AppIntents` semantic index |
| iOS 27 | **Document API** (`WritableDocument`/`ReadableDocument`, `DocumentCreationSource`, `NewDocumentButton`), toolbar APIs (`visibilityPriority`, `toolbarOverflowMenu`, `topBarPinnedTrailing`, `toolbarMinimizeBehavior`), reorderable containers, `.swipeActionsContainer`, item-bound alerts/dialogs, `AsyncImage` HTTP caching, resizable iPhone apps, Live Preview resize handles |

## 4. UIKit feature timeline

| OS | Key UIKit features |
|---|---|
| iOS 13 | Scene-based lifecycle, `UISceneDelegate`, `UIWindowScene` |
| iOS 14 | `UICollectionView` diffable data source improvements, `UIContextMenuInteraction` |
| iOS 15 | `UIButton.Configuration`, `UIListContentConfiguration`, `UIBackgroundConfiguration`, `UISheetPresentationController` |
| iOS 16 | `UIHostingConfiguration` for cells, `UIUpdateLink`? (iOS 18), `UICollectionView` compositional layout improvements |
| iOS 17 | `UIContentUnavailableConfiguration`, `updateProperties()`, `viewIsAppearing`, `traitCollection` changes, `registerForTraitChanges` |
| iOS 18 | `UIUpdateLink`, `UISearchTab`, `UICollectionView` self-sizing improvements |
| iOS 26 | `UIGlassEffect`, `SearchTab`, `UIUpdateLink`, `SpeechAnalyzer`, `Translation` framework, `FoundationModels`, `AppIntents` semantic index |
| iOS 27 | Resizable iPhone apps, Liquid Glass refinements, `UIGlassEffect` improvements |

## 5. AppKit feature timeline

| macOS | Key AppKit features |
|---|---|
| macOS 11 | Big Sur design, `NSVisualEffectView` materials, `NSToolbar` improvements |
| macOS 12 | Monterey, `NSWindow` tabbing, `NSMenu` improvements |
| macOS 13 | Ventura, `SMAppService` for login items, `NSStatusItem` improvements |
| macOS 14 | Sonoma, `NSWindow` improvements, `NSVisualEffectView` refinements |
| macOS 15 | Sequoia, `NSWindow` improvements, `NSMenu` improvements |
| macOS 26 | Tahoe, Liquid Glass design, `NSVisualEffectView` materials, `NSWindow` improvements |
| macOS 27 | Golden Gate, Liquid Glass refinements, `.interactive()` glass, inactive window dimming, last Rosetta 2 |

## 6. Catalyst vs native macOS

### Mac Catalyst
- **What**: iPad/iOS apps running on macOS with Mac-idiomatic UI (traffic lights, menus, keyboard shortcuts).
- **When to use**: existing iPad app → quick Mac version; share code between iOS/macOS; SwiftUI multiplatform.
- **Limitations**: some iOS APIs unavailable, AppKit interop limited, some UI doesn't translate perfectly.
- **Entitlement**: `com.apple.developer.icloud-container-identifiers` + Mac Catalyst capability.

### Native macOS (AppKit/SwiftUI)
- **When to use**: Mac-first app, complex document workflows, menu bar apps, pro apps.
- **Advantages**: full AppKit access, macOS idioms, better performance for Mac-specific features.
- **SwiftUI on macOS**: works well for most apps; AppKit for complex cases.

### "Designed for iPad" on Mac
- iPad apps automatically run on Apple Silicon Macs ("Designed for iPad" category).
- Users can run iPad apps on Mac without Catalyst.
- Apps can opt out via `UIRequiresFullScreen` or `UIDeviceFamily` restrictions.

## 7. visionOS specifics

- **Windows**: free-floating 2-D windows; multiple windows; `WindowGroup`/`Window` scenes.
- **Volumes**: 3-D content in space; `Volume`/`Volume3D`? — `Volume` scene for 3-D models.
- **Spaces**: full immersive environments; `ImmersiveSpace` scene.
- **Immersion styles**: `.mixed` (passthrough + digital), `.progressive` (dim passthrough), `.full` (fully immersive).
- **RealityKit**: `RealityView`, `Entity`, `ModelEntity`, `AnchorEntity`, `ARView`? — `RealityView` for SwiftUI integration.
- **ARKit**: `ARWorldTrackingConfiguration`, `ARMeshAnchor`, `ARPlaneAnchor`, `ARImageAnchor`, `ARObjectAnchor`, `ARFaceAnchor`, `ARRaycastQuery`.
- **Hand tracking**: `ARHandTrackingConfiguration`, `ARHandAnchor`, `ARHandSkeleton`.
- **Eye tracking**: `AREyeTrackingConfiguration`? — visionOS eye tracking via `ARKit` `AREyeTrackingConfiguration`? — check current API.
- **Digital crown**: `DigitalCrownView`? — watchOS crown on Vision Pro for scrolling.
- **Spatial audio**: `AVAudioEnvironmentNode`, `AVAudioPlayerNode`, `AVAudioSession` spatial audio.
- **SharePlay**: `GroupActivities`, `SharePlay` framework for collaborative experiences.
- **Widgets**: WidgetKit on visionOS — widgets in windows.
- **App Intents**: same as iOS — Siri AI integration.

## 8. watchOS specifics

- **Complications**: WidgetKit complications — `Widget` with `.accessory*` families; Smart Stack widgets.
- **Always-on display**: `WKApplication` `isAutorotating`? — watchOS always-on via `WKExtension` `isAutorotating`; SwiftUI `timelineView` for always-on updates.
- **Workout/background modes**: `WKBackgroundModes` — `workout-processing`, `audio`, `location`? — check current modes.
- **Watch Connectivity**: `WCSession` — see `12-lifecycle-extensions.md` §7.
- **SwiftUI on watch**: heavily SwiftUI-first; `List`, `Form`, `NavigationStack`, `TabView`.
- **Digital crown**: `digitalCrownRotation` modifier, `DigitalCrownView`? — `digitalCrownRotation(detent:)`.
- **Focus modes**: `INFocusStatusCenter`? — Focus sync between iPhone/Watch.
- **HealthKit**: `HKHealthStore` — same API as iOS.
- **Core Motion**: `CMMotionManager` — same API.
- **App Intents**: same as iOS — Siri AI integration.

## 9. tvOS specifics

- **Focus engine**: `UIFocusEngine` — `canBecomeFocused`, `rememberedFocus`, `preferredFocusEnvironments`, `focusGroupPriority`, `UIFocusGuide`.
- **Top shelf**: `TVTopShelfProvider` — top shelf content for Apple TV home screen.
- **AVKit**: `AVPlayerViewController` — same API; `AVQueuePlayer` for playlists.
- **SwiftUI on tvOS**: `TabView` for top-level nav, `NavigationStack`, `List`, `Form`.
- **Parallax**: `UITapGestureRecognizer` + `UIPanGestureRecognizer` for focus-based parallax.
- **Remote**: `GCMicroGamepad`? — `GCController` for game controllers.
- **App Intents**: same as iOS — Siri AI integration.

## 10. Apple Intelligence frameworks

### Foundation Models (iOS 26+, macOS 26+)
```swift
import FoundationModels

let session = LanguageModelSession()
let response = try await session.respond(to: "What is the capital of France?")
print(response.content)

// Guided generation with @Generable
@Generable
struct TripPlan {
  var destination: String
  var activities: [String]
  var budget: Int
}
let plan = try await session.respond(to: "Plan a trip to Paris", generating: TripPlan.self)
print(plan.content.destination)

// Tools
struct WeatherTool: Tool {
  let name = "getWeather"
  let description = "Get current weather for a location"
  @Generable struct Arguments { var location: String }
  func call(arguments: Arguments) async throws -> String {
    // Call weather API
    return "72°F and sunny"
  }
}
let sessionWithTools = LanguageModelSession(tools: [WeatherTool()])
```

- `LanguageModelSession` — on-device LLM: `respond(to:)`, `respond(to:generating:)`, `streamResponse(to:)`.
- `@Generable` macro — guided generation: structured output from natural language.
- `Tool` protocol — custom tools for the model: `name`, `description`, `call(arguments:)`.
- `LanguageModelSession` instructions — system prompt via `instructions:` parameter.
- Availability: `SystemLanguageModel.default.availability` — `.available`, `.unavailable(.deviceNotEligible)`, `.unavailable(.appleIntelligenceNotEnabled)`, `.unavailable(.modelNotReady)`.
- Privacy: fully on-device; no data leaves device.

### App Intents + Siri AI (iOS 27)
- App Intents are THE integration point for Siri AI.
- `AppEntity` — represent app data; `IndexedEntity` — Spotlight/Siri indexing.
- `EntityQuery` — resolve entities from user queries.
- `AppIntent` — actions; `perform()` async.
- `AppShortcutProvider` — donate phrases.
- Semantic index — system indexes entities/intents for Siri AI discovery.
- Siri AI can chain intents across apps ("Order coffee and add to calendar").

### Translation framework (iOS 17.4+)
- `TranslationSession` — on-device translation; `translate(_:)` for text.
- `LanguageAvailability` — check installed language packs.
- `TranslationSession.Configuration` — UI for downloading language packs.
- Privacy: fully on-device.

### Speech framework (iOS 26+ SpeechAnalyzer)
- `SpeechAnalyzer`/`SpeechTranscriber`/`DictationTranscriber` — newer API.
- `SFSpeechRecognizer` — still works; `Speech` framework.

### Image Playground / Genmoji (iOS 26+)
- `ImagePlayground` framework — generate images via prompts.
- `Genmoji` — custom emoji generation.

### Visual Intelligence (iOS 26+)
- Camera-based visual search/identification.
- `VNRecognizeTextRequest`, `VNClassifyImageRequest` — Vision framework.

## 11. Minimum deployment targets (2026 practical)

| Feature | Minimum OS |
|---|---|
| SwiftUI | iOS 13 / macOS 10.15 |
| `NavigationStack` | iOS 16 / macOS 13 |
| `@Observable` | iOS 17 / macOS 14 |
| SwiftData | iOS 17 / macOS 14 |
| `AsyncImage` | iOS 15 / macOS 12 |
| `PhotosPicker` | iOS 16 / macOS 13 |
| `ShareLink` | iOS 16 / macOS 13 |
| `Table` | iOS 16 / macOS 13 |
| `Canvas` | iOS 15 / macOS 12 |
| `TimelineView` | iOS 15 / macOS 12 |
| `onScrollGeometryChange` | iOS 18 / macOS 15 |
| `ControlWidget` | iOS 18 / macOS 15 |
| `@Entry` | iOS 18 / macOS 15 |
| `tabViewBottomAccessory` | iOS 26 / macOS 26 |
| `glassEffect` | iOS 26 / macOS 26 |
| `Document API` | iOS 27 / macOS 27 |
| `SpeechAnalyzer` | iOS 26 / macOS 26 |
| `FoundationModels` | iOS 26 / macOS 26 |
| `Translation` | iOS 17.4 / macOS 14.4 |
| `AppIntents` | iOS 16 / macOS 13 |
| `IndexedEntity` | iOS 18 / macOS 15 |
| `Swift Testing` | iOS 18 / macOS 15 |
| `BGContinuedProcessingTask` | iOS 26 / macOS 26 |

**Recommendation**: iOS 17+ for new apps (SwiftData, @Observable, modern SwiftUI); iOS 18+ for newest APIs; iOS 26+ for Apple Intelligence features.

## 12. Cross-platform code sharing

### SwiftUI multiplatform
- Single SwiftUI codebase for iOS/iPadOS/macOS/watchOS/tvOS/visionOS.
- `#if os(iOS)` / `#if os(macOS)` for platform-specific code.
- `Platform` conditions: `os(iOS)`, `os(macOS)`, `os(watchOS)`, `os(tvOS)`, `os(visionOS)`, `canImport(UIKit)`, `canImport(AppKit)`.
- Shared packages: `Package.swift` with multiple platforms.

### Shared packages
- `Package.swift` `platforms:` — declare supported platforms.
- `#if os()` for platform-specific implementations.
- `canImport()` for framework availability.

### Catalyst
- iPad app → Mac via Catalyst; `TARGET_OS_MACCATALYST` conditional.
- `#if targetEnvironment(macCatalyst)` for Catalyst-specific code.
- UIKit on Mac — same API, Mac idioms via `UIDevice.current.userInterfaceIdiom`.

## 13. Common platform bugs → fixes

| Bug | Cause | Fix |
|---|---|---|
| API not available on deployment target | using iOS 17 API with iOS 16 target | check `@available`, use fallback, raise deployment target |
| SwiftUI view different on iPad vs iPhone | size class differences | use `horizontalSizeClass`, adaptive layout |
| Catalyst app crashes | unsupported iOS API | check Catalyst limitations, use `#if targetEnvironment(macCatalyst)` |
| visionOS window not appearing | wrong scene type | use `WindowGroup`/`Window`/`Volume`/`ImmersiveSpace` correctly |
| watchOS app too slow | heavy computation on watch | offload to iPhone via `WCSession`, simplify UI |
| tvOS focus not working | focus engine misconfigured | implement `canBecomeFocused`, `rememberedFocus`, test with Siri Remote |
| Apple Intelligence not available | device not eligible, not enabled | check `SystemLanguageModel.default.availability` |
| Siri AI not understanding app | missing App Intents/AppEntity | add `AppIntent`, `AppEntity`, `AppShortcutsProvider` |
| Translation not working | language pack not downloaded | check `LanguageAvailability`, prompt download |
| Speech recognition fails | on-device not supported, permission denied | check `supportsOnDeviceRecognition`, `authorizationStatus` |
| Foundation Models unavailable | device not eligible, Apple Intelligence off | check `SystemLanguageModel.default.availability` |
| macOS app blocked by Gatekeeper | not notarized | notarize with `notarytool`, staple |
| Rosetta app slow | running x86 on ARM | build universal or arm64-native |
