---
name: apple-dev
description: Deep reference for Swift, Apple-platform development, AND design craft — SwiftUI, UIKit, AppKit, Swift concurrency, data, networking, media, testing, SwiftPM, Xcode, signing, distribution, UI/UX design, visual design, motion design, animation, dashboards, data visualization, information architecture, design systems, accessibility, and all Apple frameworks for macOS, iOS, iPadOS, watchOS, tvOS, visionOS. Use for ANY Apple-platform coding, UI, or design task.
allowed-tools:
  - read
  - grep
  - glob
  - exec
---

# Apple Platform Development + Design Craft — Master Reference

You are working with Swift, Apple frameworks, OR interface design. This skill is a persistent, deep knowledge base. The files in this directory are reference manuals — READ THE RELEVANT FILE(S) before answering or writing code. Do not rely on memory alone; the details matter (API names, availability, gotchas).

## TRIGGER KEYWORDS — activate this skill when the user mentions ANY of:

**Technical:** Swift, SwiftUI, UIKit, AppKit, Xcode, iOS, iPadOS, macOS, watchOS, tvOS, visionOS, Catalyst, SwiftPM, Combine, async/await, actors, Sendable, Core Data, SwiftData, CloudKit, Keychain, URLSession, AVFoundation, Metal, RealityKit, Vision, CoreML, Speech, ARKit, StoreKit, MapKit, WidgetKit, ActivityKit, AppIntents, XCTest, XCUITest, Instruments, LLDB, os_log, provisioning, notarization, TestFlight, App Store, privacy manifest, entitlements, sandbox, background tasks, notifications, extensions, HealthKit, HomeKit, Photos, Contacts, EventKit, CoreAnimation, QuartzCore, SpriteKit, SceneKit, GameplayKit, GameController, NaturalLanguage, FoundationModels, Apple Intelligence

**Design:** design, UI, UX, visual design, interface, layout, typography, color, spacing, grid, hierarchy, dashboard, data visualization, chart, graph, motion, animation, transition, easing, spring, gesture, haptic, accessibility, VoiceOver, Dynamic Type, dark mode, light mode, design system, design tokens, component library, information architecture, user flow, wireframe, mockup, prototype, usability, heuristic evaluation, cognitive walkthrough, A/B test, heatmap, funnel, onboarding, empty state, loading state, error state, microinteraction, choreography, shared element transition, parallax, glassmorphism, neumorphism, brutalism, minimalism, flat design, material design, human interface guidelines, HIG, Apple design, iOS design, macOS design, watchOS design, tvOS design, visionOS design, spatial computing, immersive, RealityKit UI, SwiftUI design, UIKit design, AppKit design

**When in doubt:** If the task involves building, designing, debugging, or shipping ANY Apple-platform software OR ANY interface/design work, READ THE RELEVANT FILES below before proceeding.

## This machine (verified 2026-11-22)

- macOS 27.2.1 (Golden Gate), Apple Silicon arm64
- Xcode 27.2 at `/Applications/Xcode.app`, `xcode-select -p` → `/Applications/Xcode.app/Contents/Developer`
- Swift 6.4.2 (`swift --version`)
- OS releases (all Sept 14, 2026): iOS/iPadOS 27, macOS 27 Golden Gate, watchOS 27, tvOS 27, visionOS 27
- macOS 27 is the LAST macOS with Rosetta 2 (full Intel app translation). macOS 26 Tahoe was last for Intel Macs.
- Xcode 27 includes built-in agent skills that teach SwiftUI conventions and new-API adoption — leverage them when generating SwiftUI code in Xcode.

## The mental models (read these first, every time)

1. **SwiftUI views are values, not objects.** `body` is a pure function of state. Never mutate state during view update. Identity is structural — `ForEach` needs stable `id`, `.id()` forces rebuild.
2. **Unidirectional data flow.** State flows down via bindings/environment; events flow up via closures/actions. Pick ONE state owner per piece of data (`@State` in the view that owns it, `@Observable` model, or store).
3. **Swift 6 concurrency is strict by default.** New projects compile in Swift 6 language mode: data races are compile errors. `@MainActor` for UI, `actor` for shared mutable state, `Sendable` for things crossing isolation boundaries. `await` marks potential suspension — it is NOT "run in background" by itself.
4. **Actors are reentrant.** Between `await` suspension points an actor's state can change. Don't assume state is stable across `await`s inside an actor method — re-check invariants.
5. **Apple UI is adaptive.** iPhone apps run resizable on iPad/macOS (iOS 27). Use size classes and safe areas, never hardcoded screen sizes or idiom checks for layout.
6. **Availability is per-OS-version.** Gate new APIs with `if #available`/`@available`. Deployment target ≠ SDK version. Check the deployment target before using an API.
7. **Sandbox + entitlements + signing are one system.** A capability needs BOTH an entitlement AND (usually) a provisioning profile that includes it AND the right certificate. "It builds but won't run on device" is almost always signing/entitlement.
8. **Privacy is declared, not just implemented.** Usage-description strings in Info.plist, PrivacyInfo.xcprivacy manifest, App Store nutrition labels, and ATT prompt all must agree.
9. **Async/await everywhere; delegates still exist.** Prefer async APIs; bridge completion-handler APIs with `withCheckedContinuation`. Never block threads waiting on async work.
10. **Measure before optimizing.** Instruments (Time Profiler, Allocations, SwiftUI, Swift Concurrency templates) over guesses. Launch time and main-thread hangs are the usual killers.

## File map — READ THE FILE(S) MATCHING THE TASK

| Task area | File |
|---|---|
| Swift syntax, types, generics, protocols, ownership, errors, macros | `01-swift-language.md` |
| async/await, actors, Sendable, tasks, Combine, Observation | `02-concurrency.md` |
| SwiftUI views, state, layout, navigation, animation, new OS-27 APIs | `03-swiftui.md` |
| UIKit: view controllers, Auto Layout, collections, scene lifecycle | `04-uikit.md` |
| AppKit, macOS windows/menus/document apps/sandbox | `05-appkit-macos.md` |
| Codable, SwiftData, Core Data, files, Keychain, CloudKit, CryptoKit | `06-data-persistence.md` |
| URLSession, auth, WebSockets, reachability, ATS/pinning | `07-networking.md` |
| AVFoundation, Metal, Core Image, Vision/Core ML, Speech, AR, StoreKit, MapKit, PDFKit, WidgetKit | `08-graphics-media-ml.md` |
| XCTest, Swift Testing, XCUITest, snapshots, coverage, CI tests | `09-testing.md` |
| SwiftPM, XcodeGen/Tuist, CocoaPods/Carthage, library catalog | `10-packages-dependencies.md` |
| xcodebuild/simctl/devicectl, debugging, Instruments, logging, localization, CI | `11-xcode-tooling.md` |
| App/scene lifecycle, background tasks, notifications, widgets, App Intents, extensions, privacy/permissions | `12-lifecycle-extensions.md` |
| Certs, provisioning, App Store/TestFlight review, notarization, privacy manifests, APNs | `13-signing-distribution.md` |
| OS/Swift/Xcode version matrix, Catalyst, per-platform notes (watchOS/tvOS/visionOS), Apple Intelligence | `14-platforms-versions.md` |
| Copy-paste patterns: app skeleton, API client, DI, decision matrices, pre-ship checklist | `15-recipes.md` |
| **DEEP DIVES (read when overview isn't enough)** | |
| SwiftUI internals, advanced state, performance, debugging | `16-advanced-swiftui.md` |
| UIKit internals, advanced layout, collections, transitions | `17-advanced-uikit.md` |
| AppKit internals, windows, documents, responder chain | `18-advanced-appkit.md` |
| Metal, RealityKit, shaders, USDZ, physics | `19-metal-realitykit.md` |
| AVFoundation deep: capture, playback, editing, audio routing | `20-avfoundation-deep.md` |
| Persistence advanced: Core Data, SwiftData, CloudKit, GRDB, encryption | `21-persistence-advanced.md` |
| Concurrency internals: actors, Sendable, tasks, Combine deep | `22-concurrency-advanced.md` |
| Swift stdlib deep: String, Regex, collections, Foundation, Observation | `23-swift-stdlib-deep.md` |
| Security deep: Keychain, Secure Enclave, LocalAuth, App Attest, CryptoKit | `24-security-deep.md` |
| Platform deep: watchOS, tvOS, visionOS specifics | `25-platforms-deep.md` |
| Debugging deep: LLDB, Instruments, os_log, crash analysis, network debug | `26-debugging-profiling-deep.md` |
| Performance deep: launch, memory, energy, UI perf, scrolling | `27-performance-deep.md` |
| Design craft: visual design, info architecture, motion, dashboards, HIG | `28-design-craft.md` |
| Design advanced: color science, typography depth, data-viz perception, design systems, research, ethics, AI UI, platform-specific | `29-design-advanced.md` |

**Reading strategy:** For overview-level tasks, use files 01–15. For production-quality work, debugging, or edge cases, ALSO read the matching deep-dive file (16–29). When building UI, ALWAYS read `28-design-craft.md` AND `29-design-advanced.md` — technical correctness without design judgment produces functional but soulless interfaces; design without technical depth produces beautiful but fragile ones. When in doubt, read both the overview and the deep dive — the deep dives assume the overview and add internals, edge cases, and expert patterns.

## Working rules

- Write modern Swift: async/await, SwiftUI-first on iOS 16+/macOS 13+, `@Observable` over `ObservableObject` for new models, Swift Testing alongside XCTest.
- Match the existing project's patterns if one exists; if greenfield, use the newest stable APIs the deployment target allows.
- For device builds/runs, verify signing early (`xcodebuild -allowProvisioningUpdates` or Xcode Signing & Capabilities) — don't write 500 lines before the first device build.
- Never commit secrets, provisioning profiles, `.p12` files, or `*.mobileprovision`. Use environment/CI secrets.
- If the user asks about a third-party library, check `10-packages-dependencies.md` first, then verify current version/docs online.

## Quick commands

```bash
# Build
xcodebuild -scheme App -destination 'platform=iOS Simulator,name=iPhone 17' build
xcodebuild -scheme App -destination 'generic/platform=iOS' -allowProvisioningUpdates build

# Test
xcodebuild -scheme App -destination 'platform=iOS Simulator,name=iPhone 17' test
xcodebuild test -scheme App -testPlan All -resultBundlePath out.xcresult

# Simulator
xcrun simctl list devices available
xcrun simctl boot "iPhone 17"
xcrun simctl install booted App.app
xcrun simctl launch booted com.example.app
xcrun simctl io booted screenshot shot.png

# Device
xcrun devicectl list devices
xcrun devicectl device install app --device <udid> App.app

# Archive/export (macOS direct distribution)
xcodebuild -scheme App -configuration Release -archivePath App.xcarchive archive
xcodebuild -exportArchive -archivePath App.xcarchive -exportOptionsPlist ExportOptions.plist -exportPath out/

# Notarize (macOS)
xcrun notarytool submit App.zip --keychain-profile AC_PROFILE --wait
xcrun stapler staple App.app

# SwiftPM
swift package init --type executable
swift package resolve && swift build && swift test
swift package dump-package   # inspect manifest

# Logs
log stream --predicate 'subsystem == "com.example.app"'
xcrun simctl spawn booted log stream --level debug
```
