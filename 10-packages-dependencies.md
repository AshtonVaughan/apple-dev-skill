# Packages, Dependencies & Project Generation

SwiftPM (standard), XcodeGen/Tuist (project generation), CocoaPods/Carthage (legacy), library catalog by category.

## 1. Swift Package Manager (SwiftPM)

### Package.swift manifest

```swift
// swift-tools-version: 6.0
import PackageDescription

let package = Package(
  name: "MyPackage",
  platforms: [.iOS(.v17), .macOS(.v14), .watchOS(.v10), .tvOS(.v17), .visionOS(.v1)],
  products: [
    .library(name: "MyLib", targets: ["MyLib"]),
    .executable(name: "MyTool", targets: ["MyTool"]),
    .library(name: "MyLibDynamic", type: .dynamic, targets: ["MyLib"])
  ],
  dependencies: [
    .package(url: "https://github.com/Alamofire/Alamofire", from: "5.9.0"),
    .package(url: "https://github.com/onevcat/Kingfisher", .upToNextMajor(from: "7.12.0")),
    .package(url: "https://github.com/pointfreeco/swift-snapshot-testing", branch: "main"),
    .package(path: "../LocalPackage")   // local path dependency
  ],
  targets: [
    .target(
      name: "MyLib",
      dependencies: [
        .product(name: "Alamofire", package: "Alamofire"),
        .product(name: "Kingfisher", package: "Kingfisher")
      ],
      path: "Sources/MyLib",
      exclude: ["Tests"],
      resources: [.process("Resources")],   // .copy for verbatim
      swiftSettings: [.swiftLanguageMode(.v6)],
      linkerSettings: [.linkedFramework("UIKit")]
    ),
    .testTarget(
      name: "MyLibTests",
      dependencies: ["MyLib", .product(name: "SnapshotTesting", package: "swift-snapshot-testing")],
      path: "Tests/MyLibTests"
    ),
    .executableTarget(
      name: "MyTool",
      dependencies: ["MyLib"],
      path: "Sources/MyTool"
    ),
    .macro(
      name: "MyMacros",
      dependencies: [
        .product(name: "SwiftSyntax", package: "swift-syntax"),
        .product(name: "SwiftSyntaxMacros", package: "swift-syntax")
      ],
      path: "Sources/MyMacros"
    )
  ]
)
```

### Key manifest concepts

- **Tools version**: `// swift-tools-version: X.Y` — determines API availability; use latest (6.0+).
- **Products**: `.library` (static by default; `.dynamic` for dylib), `.executable`, `.plugin`.
- **Targets**: `.target` (source), `.testTarget`, `.executableTarget`, `.macro` (macro implementation), `.systemLibrary` (C system libs), `.binaryTarget` (prebuilt .xcframework).
- **Dependencies**: `.package(url:from:)` (semver), `.package(url:_:)` (branch/revision/exact), `.package(path:)` (local), `.package(url: .upToNextMinor/.upToNextMajor/.exact)`, `.package(url:branch:)`, `.package(url:revision:)`, `.package(url:from:)`.
- **Products**: `.product(name:package:)` — link specific product from dependency.
- **Resources**: `.process` (Xcode processes/optimizes), `.copy` (verbatim copy), `.embedInCode`? — `.process` is standard.
- **Swift settings**: `.swiftLanguageMode(.v5/.v6)`, `.unsafeFlags`, `.define`, `.enableExperimentalFeature`, `.enableUpcomingFeature`, `.strictMemorySafety`, `.strictConcurrency`.
- **C/C++ interop**: `.interoperabilityMode(.Cxx)` for C++ targets.
- **Binary targets**: `.binaryTarget(name:url:)` — prebuilt `.xcframework`/`.a`/`.so`.
- **System library targets**: `.systemLibrary` — wrap system C libraries.
- **Macro targets**: `.macro` — compiler plugins for code generation; dependencies on `swift-syntax`.
- **Plugin targets**: `.plugin` — build tool plugins (codegen, resource processing).

### SwiftPM commands

```bash
swift package init --type library|executable|tool
swift package init --name MyPackage --type library
swift package edit <package-name>          # checkout for local modification
swift package unedit <package-name>
swift package resolve                        # resolve dependencies → Package.resolved
swift package update                         # update to latest allowed versions
swift package update <package-name>          # update specific
swift package show-dependencies              # tree view
swift package dump-package                   # resolved manifest JSON
swift package describe                       # package description
swift package clean                          # clean build artifacts
swift package reset                          # reset to resolved state
swift build                                  # build
swift build -c release                       # release build
swift build --product MyTool                 # build specific product
swift test                                   # run tests
swift test --filter MyTests/testName         # specific test
swift test --parallel                         # parallel testing
swift run MyTool                             # run executable
swift run                                    # run default executable
swift package --disable-sandbox              # for plugins needing network
```

- `Package.resolved` — pins exact versions/commits; commit this file for reproducible builds.
- `swift build` uses `.build/` directory; `swift test` builds test targets.
- **Swift Build** (Swift 6.4+) — new default build system replacing SwiftPM's internal build; same build graph across platforms; `swift package` commands use it automatically.
- `swift package edit` — checks out dependency source for local modification; `unedit` returns to resolved state.
- Binary artifacts: `.binaryTarget` downloads `.xcframework` on first build; cached in `~/.swiftpm` or `.build`.

### Local packages

- `Package(path: "../LocalPackage")` — local path dependency; edits reflect immediately (no version pinning).
- `Package(url: "file:///path/to/package")` — local git repo.
- Best practice: local packages for code sharing within monorepo; remote for external deps.

## 2. XcodeGen

Project generation from YAML spec — replaces `.xcodeproj` in version control.

```yaml
# project.yml
name: MyApp
options:
  bundleIdPrefix: com.example
  deploymentTarget:
    iOS: "17.0"
    macOS: "14.0"
targets:
  MyApp:
    type: application
    platform: iOS
    sources:
      - path: Sources
        excludes: ["**/*.md"]
    dependencies:
      - package: Alamofire
      - target: MyAppKit
    settings:
      base:
        PRODUCT_BUNDLE_IDENTIFIER: com.example.myapp
        SWIFT_VERSION: "6.0"
        GENERATE_INFOPLIST_FILE: YES
        INFOPLIST_KEY_UILaunchScreen_Generation: YES
        INFOPLIST_KEY_UISupportedInterfaceOrientations: "UIInterfaceOrientationPortrait"
    scheme:
      testTargets: [MyAppTests]
  MyAppKit:
    type: framework
    platform: iOS
    sources: [SourcesKit]
  MyAppTests:
    type: bundle.unit-test
    platform: iOS
    sources: [Tests]
    dependencies: [target: MyApp]
packages:
  Alamofire:
    url: https://github.com/Alamofire/Alamofire
    version: 5.9.0
schemes:
  MyApp:
    build:
      targets:
        MyApp: all
        MyAppTests: [test]
    test:
      targets: [MyAppTests]
```

```bash
xcodegen generate              # generates .xcodeproj from project.yml
xcodegen --spec project.yml    # custom spec
xcodegen dump                  # JSON spec
```

- Pros: human-readable YAML, git-friendly diffs, no `.pbxproj` merge conflicts, template support.
- Cons: another tool dependency, XcodeGen version must support latest Xcode features.
- `project.yml` → `xcodegen generate` → `.xcodeproj` (gitignore generated project or commit it? — typically gitignore and regenerate in CI).

## 3. Tuist

More powerful project generation — cache, binary targets, remote cache, project description in Swift.

```swift
// Project.swift
import ProjectDescription

let project = Project(
  name: "MyApp",
  targets: [
    .target(name: "MyApp", destinations: [.iPhone, .iPad], product: .app,
      bundleId: "com.example.myapp",
      deploymentTargets: .iOS("17.0"),
      sources: ["Sources/**"],
      dependencies: [
        .external(name: "Alamofire"),
        .target(name: "MyAppKit")
      ])
  ],
  schemes: [
    .scheme(name: "MyApp", shared: true, buildAction: .buildAction(targets: [.target("MyApp")]),
      testAction: .targets([.testableTarget(target: "MyAppTests")]))
  ]
)
```

```bash
tuist init                         # create Tuist project
tuist generate                    # generate .xcodeproj(s)
tuist install                     # install dependencies (binary cache)
tuist build                       # build via Tuist
tuist test                        # test via Tuist
tuist cache                        # manage binary cache
tuist edit                        # open in Xcode for editing
```

- **Tuist** advantages over XcodeGen: binary cache (prebuilt frameworks — faster CI), remote cache, project graph analysis, Swift-based DSL, modular architecture support, `Tuist.swift` for dependencies.
- `Tuist.swift` — dependency manifest: `.remote(url:version:)`, `.local(path:)`.
- Binary cache: `tuist cache` stores built frameworks; CI restores cache → faster builds.
- `tuist edit` — opens Xcode for manual project editing (generates workspace).
- Migration: Tuist 4.x uses `Tuist.swift` + `Project.swift` per module; `tuist generate` creates workspace.

## 4. CocoaPods (legacy)

```ruby
# Podfile
platform :ios, '17.0'
use_frameworks!
inhibit_all_warnings!

target 'MyApp' do
  pod 'Alamofire', '~> 5.9'
  pod 'Kingfisher', '~> 7.12'
  pod 'SnapKit', '~> 5.7'
end

target 'MyAppTests' do
  inherit! :search_paths
end

post_install do |installer|
  installer.pods_project.targets.each do |target|
    target.build_configurations.each do |config|
      config.build_settings['IPHONEOS_DEPLOYMENT_TARGET'] = '17.0'
    end
  end
end
```

```bash
pod install          # installs pods → MyApp.xcworkspace (USE THIS, not .xcodeproj)
pod update           # update pods
pod outdated         # check for updates
pod deintegrate      # remove CocoaPods from project
```

- **CRITICAL**: after `pod install`, open `.xcworkspace` NOT `.xcodeproj` — pods are in workspace.
- `Podfile.lock` — pins versions; commit it.
- `use_frameworks!` — dynamic frameworks (required for Swift pods); `use_modular_headers!` for static.
- `inherit! :search_paths` — test targets inherit pod search paths.
- `post_install` hook — fix build settings (deployment target, architectures).
- Source: `source 'https://cdn.cocoapods.org/'` or private specs repo.
- **Status**: still works but declining; SPM is standard for new projects. CocoaPods for legacy deps without SPM support.

## 5. Carthage (legacy)

```bash
# Cartfile
github "Alamofire/Alamofire" ~> 5.9
github "onevcat/Kingfisher" ~> 7.12
```

```bash
carthage update --platform iOS     # fetch + build frameworks → Carthage/Build/
carthage bootstrap --platform iOS  # build from Cartfile.resolved
```

- Drag `Carthage/Build/iOS/Alamofire.framework` into Xcode project.
- Build phase: "Run Script" → `/usr/local/bin/carthage copy-frameworks` with `$(SRCROOT)/Carthage/Build/iOS/*.framework` in input files.
- `Cartfile.resolved` — pins versions; commit.
- **Status**: largely deprecated; SPM preferred. Still used for some binary frameworks.

## 6. Library catalog by category

### Networking
- **Alamofire** — HTTP client: interceptors, retry, multipart, validation, async/await.
- **Moya** — Alamofire-based API abstraction: TargetType protocol, type-safe endpoints.
- **Apollo iOS** — GraphQL client: codegen, normalized cache, SwiftUI integration.
- **Network.framework** — native: `NWConnection`, `NWListener`, `NWBrowser`, `NWPathMonitor`.

### Images
- **Kingfisher** — image loading/caching: `KFImage`, memory+disk cache, processors, transitions.
- **Nuke** — modern Swift image loading: `LazyImage`, `ImagePipeline`, fast, async-first.
- **SDWebImageSwift** — popular alternative; UIKit-bridged.
- **NukeUI** — SwiftUI helpers for Nuke.

### UI/Layout
- **SnapKit** — Auto Layout DSL: `snp.makeConstraints`, `snp.updateConstraints`, readable syntax.
- **TinyConstraints** — minimal Auto Layout DSL.
- **Stevia** — layout DSL: `stack`, `label`, `button` chains.
- **SwiftUI** — native declarative UI (preferred for new code).
- **DGCharts** (danielgindi/Charts) — UIKit charts: line/bar/pie/radar/candle.

### Reactive/State
- **Combine** — Apple framework: publishers, subjects, operators, schedulers.
- **RxSwift/RxCocoa** — ReactiveX for Swift: observables, subjects, operators, RxCocoa UIKit bindings. (Declining but still used.)
- **ReactiveSwift** — ReactiveCocoa successor; less common now.
- **@Observable** — Apple Observation framework: fine-grained tracking for SwiftUI.
- **TCA** (The Composable Architecture) — Point-Free architecture: unidirectional data flow, testable, SwiftUI-first.

### Dependency Injection
- **Swinject** — DI container: `Assembler`, `Container`, `ServiceEntry`.
- **Factory** — modern DI: `@Injected`, `Container.shared`, property wrappers.
- **Resolver** — lightweight DI.
- **Needle** — compile-time DI (Uber): generated components, no runtime reflection.
- **Dependencies** (Point-Free) — `@Dependency` property wrapper, test overrides.

### Persistence
- **GRDB.swift** — SQLite wrapper: type-safe queries, migrations, observation.
- **SQLite.swift** — lightweight SQLite wrapper.
- **Realm** — object database (MongoDB): `@Persisted`, `Results`, Atlas sync.
- **SwiftData** — Apple native ORM (preferred for new code).
- **CoreStore** — Core Data wrapper with better API.

### Keychain
- **KeychainAccess** — simple Keychain wrapper: `Keychain(service:).accessibility`, subscript access.
- **Valet** — Square's Keychain wrapper: `Valet.keychain`, migration support.

### Logging
- **os_log/Logger** — Apple unified logging (native, preferred).
- **SwiftyBeaver** — file+console logging with log levels, remote logging.
- **Willow** (Point-Free) — structured logging with privacy, redaction.

### Utilities
- **Then** — `then`/`with`/`tap` helpers for cleaner init chains.
- **Defaults** (sindresorhus) — type-safe UserDefaults wrapper with property wrappers.
- **DifferenceKit** — diffing for collection view updates.
- **swift-algorithms** (Apple) — `chain`, `chunked`, `combinations`, `permutations`, `windows`, `adjacentPairs`, `trimming`, `unique`.
- **swift-collections** (Apple) — `Deque`, `OrderedSet`, `OrderedDictionary`, `Heap`, `Tree`.
- **swift-numerics** — `Complex`, `SIMD` types, numeric algorithms.
- **swift-argument-parser** — CLI argument parsing for executables.
- **swift-log** — logging API (used by server-side Swift).
- **swift-async-algorithms** — `AsyncTimerSequence`, `AsyncDebounceSequence`, `AsyncChannel`, `AsyncThrowingChannel`.
- **CollectionConcurrencyKit** — `asyncMap`, `asyncCompactMap`, `asyncFilter` for collections.
- **SwiftProtobuf** — Protocol Buffers support.

### Lottie/Animation
- **lottie-ios** (Airbnb) — Lottie animations: `LottieAnimationView`, `.loop`, `.play`, dotLottie support.
- **Rive** — interactive animations/state machines: `RiveViewModel`, state machine inputs.
- **Spring** (Jasdev?) — spring animation helpers: `SpringAnimation`, `SpringTiming`.

### Markdown/Formatting
- **MarkdownUI** (gonzalezreal) — Markdown rendering: `Markdown`, `MarkdownView`, themes, image providers.
- **SwiftRichString** — attributed string builder.
- **AttributedString** — native markdown/HTML parsing (iOS 15+).

### Permissions
- **Permission** (delightroom) — unified permission API: `Permission.camera.status`, `.request`.
- **SPPermissions** — permission management UI.

### Device/Info
- **DeviceKit** — device info: `Device.current`, `DeviceType`, `isPad`, `systemVersion`.
- **KeychainAccess** — see Keychain section.

### Testing
- **XCTest** — native unit/UI testing.
- **Swift Testing** — modern testing framework.
- **swift-snapshot-testing** (Point-Free) — snapshot tests for views.
- **ViewInspector** — inspect SwiftUI view hierarchy in tests.
- **swift-custom-dump** — better `dump()` for debugging.

### Formatters/Validation
- **SwiftValidator** — input validation: `String.validate(.email)`, `.phoneNumber`.
- **PhoneNumberKit** — phone number parsing/formatting.
- **Format** — byte/file size formatting.

### Feature Flags/Analytics
- **GrowthBook** — feature flags + A/B testing SDK.
- **Statsig** — feature gates, experiments, analytics.
- **LaunchDarkly** — feature management platform.
- **Firebase** (Google) — analytics, crashlytics, messaging, remote config, auth, firestore, storage.
- **Amplitude** — product analytics.
- **Mixpanel** — event analytics.
- **PostHog** — open-source analytics.
- **TelemetryDeck** — privacy-focused analytics.
- **Sentry** — crash reporting + performance monitoring.
- **OneSignal** — push notifications.

### Paywall/Subscriptions
- **RevenueCat** (purchases-ios) — StoreKit wrapper: offerings, purchases, restore, analytics, entitlements.
- **Superwall** — paywall framework with A/B testing.

### Localization
- **SwiftGen** — code generation from localization files/xcstrings/assets: `L10n`, `Asset`, `Color`, `Image` enums.
- **Localization** macros — native `String(localized:)`, `LocalizedStringKey`.

### Other
- **Vortex** — SwiftUI architecture helpers.
- **ComposableArchitecture** (TCA) — see Reactive/State.
- **Needle** — see DI.
- **Factory** — see DI.
- **Nuke** — see Images.
- **GRDB.swift** — see Persistence.
- **KeychainAccess** — see Keychain.
- **Defaults** — see Utilities.
- **Then** — see Utilities.
- **SwiftProtobuf** — see Utilities.
- **Lottie** — see Animation.
- **MarkdownUI** — see Markdown.
- **Permission** — see Permissions.
- **DeviceKit** — see Device.
- **ViewInspector** — see Testing.
- **SnapshotTesting** — see Testing.
- **CustomDump** — see Testing.
- **PhoneNumberKit** — see Validation.
- **Format** — see Validation.
- **GrowthBook/Statsig/LaunchDarkly** — see Feature Flags.
- **Firebase/Amplitude/Mixpanel/PostHog/TelemetryDeck/Sentry/OneSignal** — see Analytics.
- **RevenueCat/Superwall** — see Paywall.
- **SwiftGen** — see Localization.
- **Vortex** — see Other.

## 7. Dependency management best practices

### Version pinning
- **Apps**: commit `Package.resolved` — exact versions for reproducible builds.
- **Libraries**: use `from:`/`upToNextMajor` for flexibility; don't commit `Package.resolved` for libraries.
- **Exact versions**: `.package(url: .exact("1.2.3"))` for critical deps.
- **Branch tracking**: `.package(url:branch:)` for development deps — risky for production.

### Security
- **Minimum release age**: prefer versions >7 days old (supply chain attacks often caught quickly).
- **Avoid `latest`/`*`**: floating versions auto-resolve to newest — pin or use semver ranges.
- **Audit dependencies**: `swift package show-dependencies`, GitHub Dependabot/Renovate for SPM.
- **License compliance**: check licenses (MIT, Apache 2.0, BSD); some require attribution.
- **Binary targets**: verify checksums; `.binaryTarget` with `.xcframework` — check signature.

### Monorepo / local packages
- Use local packages (`Package(path:)`) for internal modules — share code without versioning overhead.
- `Tuist` binary cache for faster builds in monorepos.
- `XcodeGen`/`Tuist` for project generation — no `.pbxproj` merge conflicts.

### Common issues
| Issue | Fix |
|---|---|
| `Package.resolved` merge conflict | regenerate with `swift package resolve`, commit result |
| Dependency not found | check package name in `products:`, verify URL/version |
| Circular dependency | restructure into separate packages or local packages |
| Build fails after update | pin to previous version, check changelog |
| Binary target download fails | check URL, use `.binaryTarget` with `.xcframework` |
| Macro target fails | ensure `swift-syntax` dependency, check tools version |
| Resource not found | use `.process` not `.copy` for most resources; check `Bundle.module` |
| Test target can't see internal | use `@testable import` for internal symbols |

## 8. Xcode project structure

### Standard layout
```
MyApp/
├── MyApp.xcodeproj (or .xcworkspace if using CocoaPods)
├── MyApp/
│   ├── App/ (App entry, AppDelegate, SceneDelegate)
│   ├── Features/ (feature modules)
│   │   ├── Auth/
│   │   │   ├── Views/
│   │   │   ├── ViewModels/
│   │   │   └── Models/
│   │   └── Settings/
│   ├── Core/ (shared services, utilities)
│   │   ├── Networking/
│   │   ├── Persistence/
│   │   ├── Extensions/
│   │   └── Utilities/
│   ├── Resources/ (assets, localization, plist)
│   └── Supporting Files/ (Info.plist, entitlements, xcconfig)
├── MyAppTests/
├── MyAppUITests/
├── Packages/ (local Swift packages)
└── project.yml (XcodeGen) or Tuist.swift
```

### Build settings levels
- **Project**: shared defaults (deployment target, Swift version, architectures).
- **Target**: app/framework/test specific (bundle ID, product type, dependencies).
- **Configuration**: Debug/Release overrides (optimization, debugging, assertions).
- **xcconfig files**: external build settings — `Debug.xcconfig`, `Release.xcconfig`, `Shared.xcconfig`; reference via `baseConfigurationReference`.

### Build phases
- **Sources**: compile Swift/ObjC/C/C++ files.
- **Frameworks**: link libraries/frameworks.
- **Resources**: copy bundles/assets/plists into app.
- **Copy Bundle Resources**: resources into app bundle.
- **Embed Frameworks**: embed dynamic frameworks.
- **Run Script**: custom build scripts (codegen, linting, signing).
- **Headers**: public headers for frameworks.
- **Compile Sources**: C/C++ compilation.
- **Link Binary**: linker flags.

### Entitlements & Info.plist
- `MyApp.entitlements` — capabilities: push notifications, app groups, iCloud, associated domains, etc.
- `Info.plist` — `GENERATE_INFOPLIST_FILE = YES` recommended; use `INFOPLIST_KEY_*` build settings for common keys (launch screen, orientations, usage descriptions).
- `PrivacyInfo.xcprivacy` — privacy manifest (required for some APIs/data).

### Asset catalogs
- `Assets.xcassets` — `AppIcon` (all sizes + dark/tinted variants), `AccentColor`, colors, images, symbols, data sets (ML models).
- Icon Composer (`.icon` format, iOS 18+) — layered app icons with Liquid Glass.

### Schemes
- `MyApp.xcscheme` — build/run/test/archive configuration; shared schemes in `xcshareddata/xcschemes/`.
- Multiple schemes: app, tests, benchmarks, different configs.
- `xcuserdata/` — user-specific, gitignore.
