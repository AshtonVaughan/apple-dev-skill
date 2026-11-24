# Xcode Tooling & CLI Reference

Xcode IDE essentials, command-line tools, debugging, Instruments, logging, localization, CI.

## 1. Xcode IDE essentials

### Navigator areas
- **Project navigator** (⌘1) — files/groups.
- **Source control** (⌘2) — git changes, blame, branches.
- **Find navigator** (⌘3) — search results.
- **Issue navigator** (⌘4) — warnings/errors.
- **Test navigator** (⌘5) — test results.
- **Debug navigator** (⌘6) — breakpoints, CPU/memory gauges.
- **Breakpoint navigator** (⌘7) — all breakpoints.
- **Report navigator** (⌘8) — build/test/log reports.

### Editor
- **Standard/Assistant/Version** editors — ⌘⏎ for assistant.
- **Minimap** — scroll overview.
- **Code folding** — ⌥⌘←/→.
- **Refactoring** — right-click symbol: rename, extract method/property, change signature.
- **Fix-its** — click warning/error lightbulb.
- **Documentation** — ⌥⌘? for Quick Help.
- **Live issues** — purple warnings for concurrency/runtime issues (Swift 6 strict).

### Inspectors
- **File inspector** (⌥⌘1) — target membership, location.
- **Identity inspector** (⌥⌘2) — class/module.
- **Attributes inspector** (⌥⌘3) — view attributes.
- **Size inspector** (⌥⌘4) — view size.
- **Connection inspector** (⌥⌘6) — IB connections (storyboards).
- **Build settings** — target/project level.

### Schemes & targets
- **Scheme** — build/run/test/archive config; multiple schemes per project.
- **Target** — app/framework/test/extension; each has build phases, settings, entitlements.
- **Configuration** — Debug/Release/custom; per-target overrides.
- **Destination** — simulator/device; "Any iOS Device" for generic.

### Build settings hierarchy
1. **Xcode defaults** — base values.
2. **Project settings** — shared across targets.
3. **Target settings** — app/framework specific.
4. **Configuration settings** — Debug/Release overrides.
5. **xcconfig** — external files referenced by target.

Key settings:
- `PRODUCT_BUNDLE_IDENTIFIER` — unique app ID.
- `SWIFT_VERSION` — Swift language version (5/6).
- `IPHONEOS_DEPLOYMENT_TARGET`/`MACOSX_DEPLOYMENT_TARGET` — minimum OS.
- `SWIFT_ACTIVE_COMPILATION_CONDITIONS` — `DEBUG` for debug builds.
- `SWIFT_OPTIMIZATION_LEVEL` — `-Onone` (debug), `-O` (release), `-Osize`.
- `SWIFT_COMPILATION_MODE` — `singlefile`/`wholemodule` (WMO faster release).
- `DEBUG_INFORMATION_FORMAT` — `dwarf` (debug), `dwarf-with-dsym` (release).
- `CODE_SIGN_STYLE` — `Automatic`/`Manual`.
- `CODE_SIGN_IDENTITY` — `Apple Development`/`Apple Distribution`/`iPhone Developer`/`iPhone Distribution`.
- `PROVISIONING_PROFILE_SPECIFIER` — profile name/UUID.
- `GENERATE_INFOPLIST_FILE` — `YES` for auto-generated Info.plist.
- `INFOPLIST_KEY_*` — build-setting-based Info.plist keys.
- `ASSETCATALOG_COMPILER_APPICON_NAME` — app icon name.
- `ENABLE_USER_SCRIPT_SANDBOXING` — `YES` for script sandboxing.
- `SWIFT_STRICT_CONCURRENCY` — `minimal`/`targeted`/`complete`.

## 2. Command-line tools

### xcodebuild

```bash
# Build
xcodebuild -scheme MyApp -destination 'platform=iOS Simulator,name=iPhone 17' build
xcodebuild -scheme MyApp -destination 'generic/platform=iOS' build
xcodebuild -scheme MyApp -configuration Release -destination 'generic/platform=iOS' build
xcodebuild -scheme MyApp -destination 'generic/platform=macOS' build
xcodebuild -scheme MyApp -destination 'platform=macOS' build

# Test
xcodebuild -scheme MyApp -destination 'platform=iOS Simulator,name=iPhone 17' test
xcodebuild -scheme MyApp -testPlan Full -destination 'platform=iOS Simulator,name=iPhone 17' test
xcodebuild -scheme MyApp -only-testing:MyAppTests/UserTests -destination 'platform=iOS Simulator,name=iPhone 17' test
xcodebuild -scheme MyApp -parallel-testing-enabled YES -maximum-parallel-testing-workers 4 -destination 'platform=iOS Simulator,name=iPhone 17' test

# Archive
xcodebuild -scheme MyApp -configuration Release -archivePath MyApp.xcarchive archive

# Export
xcodebuild -exportArchive -archivePath MyApp.xcarchive -exportOptionsPlist ExportOptions.plist -exportPath ./export

# Signing
xcodebuild -scheme MyApp -destination 'generic/platform=iOS' -allowProvisioningUpdates build
xcodebuild -scheme MyApp -destination 'generic/platform=iOS' -allowProvisioningDeviceRegistration build

# Clean
xcodebuild -scheme MyApp clean

# Show SDKs
xcodebuild -showsdks
xcodebuild -showdestinations -scheme MyApp

# Result bundle
xcodebuild test -scheme MyApp -resultBundlePath TestResults.xcresult
xcrun xcresulttool get test-results summary --path TestResults.xcresult
```

- `-scheme` — scheme name.
- `-destination` — `platform=iOS Simulator,name=iPhone 17`, `generic/platform=iOS`, `platform=macOS`, `id=<device-udid>`.
- `-configuration` — `Debug`/`Release`.
- `-derivedDataPath` — custom DerivedData location.
- `-archivePath` — archive output path.
- `-exportOptionsPlist` — export configuration plist.
- `-allowProvisioningUpdates` — auto-manage signing (requires Apple ID in Xcode).
- `-allowProvisioningDeviceRegistration` — register device automatically.
- `-resultBundlePath` — save test results.
- `-parallel-testing-enabled`/`maximum-parallel-testing-workers` — parallel testing.
- `-only-testing:`/`-skip-testing:` — filter tests.
- `-enableCodeCoverage` — coverage data.

### xcrun simctl (Simulator)

```bash
# List
xcrun simctl list devices                    # all devices
xcrun simctl list devices available          # available only
xcrun simctl list devices booted             # booted only
xcrun simctl list runtimes                   # iOS versions
xcrun simctl list devicetypes                # device types

# Create
xcrun simctl create "iPhone 17" com.apple.CoreSimulator.SimDeviceType.iPhone-17 com.apple.CoreSimulator.SimRuntime.iOS-27-0

# Boot/shutdown
xcrun simctl boot "iPhone 17"
xcrun simctl shutdown "iPhone 17"
xcrun simctl bootstatus "iPhone 17" -b        # wait for boot

# Install/launch
xcrun simctl install booted /path/to/App.app
xcrun simctl launch booted com.example.app    # returns PID
xcrun simctl launch booted com.example.app --args -UITestMode YES
xcrun simctl terminate booted com.example.app

# Uninstall
xcrun simctl uninstall booted com.example.app

# Files
xcrun simctl get_app_container booted com.example.app data   # app data container
xcrun simctl get_app_container booted com.example.app app    # app bundle
xcrun simctl openurl booted "myapp://path"                   # open URL

# IO
xcrun simctl io booted screenshot /tmp/screen.png
xcrun simctl io booted recordVideo /tmp/video.mp4
xcrun simctl io booted recordVideo --codec h264 /tmp/video.mp4
xcrun simctl io booted enumerate                    # list IO devices
xcrun simctl io booted screenshot --type=png --mask=alpha /tmp/screen.png

# Status bar
xcrun simctl status_bar booted override --time "9:41" --batteryState charged --batteryLevel 100 --wifiBars 3 --cellularBars 4
xcrun simctl status_bar booted clear

# Push notifications
xcrun simctl push booted com.example.app /path/to/payload.json

# Privacy permissions
xcrun simctl privacy booted grant camera com.example.app
xcrun simctl privacy booted revoke camera com.example.app
xcrun simctl privacy booted reset all com.example.app

# Erase
xcrun simctl erase "iPhone 17"

# Spawn (run commands in sim)
xcrun simctl spawn booted log stream --level debug
xcrun simctl spawn booted launchctl list
```

- `booted` — shorthand for currently booted device.
- `get_app_container` — access app sandbox files for debugging.
- `privacy` — grant/revoke permissions without UI.
- `push` — simulate push notifications with JSON payload.
- `io` — screenshot, video, enumerate hardware.

### xcrun devicectl (Physical devices)

```bash
# List
xcrun devicectl list devices                    # all devices
xcrun devicectl list devices --json-output /tmp/devices.json

# Device info
xcrun devicectl device info details --device <udid>

# Install/launch
xcrun devicectl device install app --device <udid> /path/to/App.app
xcrun devicectl device process launch --device <udid> com.example.app
xcrun devicectl device process launch --device <udid> --console com.example.app   # with console output
xcrun devicectl device process terminate --device <udid> com.example.app

# Copy files
xcrun devicectl device copy to --device <udid> --source /local/file --destination /remote/path
xcrun devicectl device copy from --device <udid> --source /remote/path --destination /local/file

# Logs
xcrun devicectl device console --device <udid>    # stream device logs
```

### Other xcrun tools

```bash
# altool (deprecated but still works for some tasks)
xcrun altool --validate-app -f App.ipa -t ios --apiKey KEY --apiIssuer ISSUER

# notarytool (macOS notarization)
xcrun notarytool submit App.zip --keychain-profile PROFILE --wait
xcrun notarytool history --keychain-profile PROFILE
xcrun notarytool log <submission-id> --keychain-profile PROFILE
xcrun stapler staple App.app
xcrun stapler validate App.app

# symbolicate crash logs
xcrun symbolicatecrash crash.ips App.app.dSYM > symbolicated.crash

# Instruments CLI
xcrun xctrace list templates
xcrun xctrace record --template 'Time Profiler' --output /tmp/trace.trace --launch -- /path/to/app
xcrun xctrace export --input /tmp/trace.trace --xpath '/trace-toc/run[@number="1"]/data/table[@schema="time-profile"]' --output /tmp/profile.xml

# lldb
xcrun lldb /path/to/binary
(lldb) process attach --pid <pid>
(lldb) process attach --name MyApp
(lldb) bt                    # backtrace
(lldb) thread backtrace all
(lldb) frame select 0
(lldb) po variable
(lldb) p variable
(lldb) expr -- variable = newValue
(lldb) memory read --size 8 --format x address
(lldb) register read
(lldb) image lookup -vn <symbol>
(lldb) breakpoint set -n functionName
(lldb) breakpoint set -S viewDidLoad
(lldb) breakpoint set -r "pattern"
(lldb) watchpoint set variable variableName
(lldb) continue
(lldb) detach
```

## 3. Debugging

### Breakpoints
- **Line breakpoint** — click gutter.
- **Exception breakpoint** — breaks on all exceptions (Objective-C/C++/Swift).
- **Symbolic breakpoint** — break on symbol: `UIViewAlertForUnsatisfiableConstraints`, `NSInternalInconsistencyException`, `objc_exception_throw`, `swift_reportError`.
- **Runtime issue breakpoint** — Main Thread Checker, Thread Sanitizer, Address Sanitizer.
- **Watchpoint** — watch variable changes.

### LLDB essentials
```lldb
# Execution
(lldb) process continue
(lldb) process interrupt
(lldb) process detach
(lldb) process kill

# Frames
(lldb) bt                          # backtrace
(lldb) frame select 1              # select frame
(lldb) frame variable              # show locals
(lldb) frame info

# Variables
(lldb) po self                     # print object (ObjC description)
(lldb) po self.property
(lldb) p variable                  # print (Swift-aware)
(lldb) v variable                  # same as p
(lldb) expr -- variable            # evaluate expression
(lldb) expr -- [self doSomething]  # call method
(lldb) memory read --size 16 --format x address
(lldb) register read rax
(lldb) register write rax 0x1234

# Images/symbols
(lldb) image list
(lldb) image lookup -vn functionName
(lldb) image lookup -a 0x12345678
(lldb) image dump symtab

# Breakpoints
(lldb) breakpoint set -n functionName
(lldb) breakpoint set -S selectorName
(lldb) breakpoint set -r "regex"
(lldb) breakpoint set -f file.swift -l 42
(lldb) breakpoint command add -o "po variable" 1
(lldb) breakpoint delete 1

# Memory
(lldb) memory region address
(lldb) memory read --count 10 --format x address
(lldb) memory write address value

# Thread
(lldb) thread backtrace all
(lldb) thread select 2
(lldb) thread info
```

### View debugging
- **Debug > View Debugger** — 3D view hierarchy, select views, see constraints, frames, properties.
- **Debug > Debug View Hierarchy** — same as above.
- **Memory Graph Debugger** — Debug > Debug Memory Graph — shows retain cycles.
- **Debug > Attach to Process** — attach to running app.
- **Debug > Simulate Location** — set GPS location.
- **Debug > Color Blended Layers** — shows offscreen rendering.
- **Debug > Color Copied Images** — shows image blending.
- **Debug > Color Misaligned Images** — shows scaled images.
- **Debug > Color Off-screen Rendered** — shows offscreen rendering.

### Runtime diagnostics
- **Main Thread Checker** — warns about UIKit/AppKit calls off main thread.
- **Thread Sanitizer** — detects data races.
- **Address Sanitizer** — detects memory errors (use-after-free, buffer overflow).
- **Undefined Behavior Sanitizer** — detects UB (signed overflow, invalid shifts).
- Enable in scheme: Product > Scheme > Edit Scheme > Run > Diagnostics.

## 4. Instruments

### Templates
- **Time Profiler** — CPU hotspots, call trees.
- **Allocations** — memory allocations, leaks.
- **Leaks** — memory leaks.
- **VM Tracker** — virtual memory regions.
- **Core Animation** — offscreen rendering, blends, images.
- **Animation Hitches** — dropped frames.
- **Hangs** — main thread hangs.
- **Swift Concurrency** — tasks, actors, priority inversions.
- **SwiftUI** — view body evaluations, updates.
- **Network** — network activity.
- **Energy Log** — energy impact.
- **File Activity** — file I/O.
- **Metal System Trace** — GPU performance.
- **Core ML** — model inference performance.
- **os_signpost** — custom intervals via `OSSignposter`.
- **Points of Interest** — `os_signpost` intervals for specific operations.
- **Time Profiler** — CPU sampling.
- **System Trace** — system-wide trace.
- **Core Data** — fetch/save performance.
- **Game Performance** — game-specific metrics.

### Usage
```bash
# CLI recording
xcrun xctrace record --template 'Time Profiler' --output /tmp/trace.trace --launch -- /path/to/app
xcrun xctrace record --template 'Allocations' --output /tmp/memory.trace --attach <pid>

# Export
xcrun xctrace export --input /tmp/trace.trace --xpath '/trace-toc/run[@number="1"]/data/table[@schema="time-profile"]' --output /tmp/profile.xml
```

- Instruments GUI: Product > Profile (⌘I) — records app, shows timeline + instruments.
- `OSSignposter` — custom intervals:
  ```swift
  import os.signpost
  let signposter = OSSignposter(subsystem: "com.example.app", category: "Networking")
  let state = signposter.beginInterval("fetchUser")
  // ... work ...
  signposter.endInterval("fetchUser", state)
  ```

## 5. Unified logging (os_log/Logger)

```swift
import os.log

// Modern Logger (iOS 14+)
let logger = Logger(subsystem: "com.example.app", category: "Networking")
logger.debug("Fetching user \(userId)")
logger.info("Request sent")
logger.notice("Cache hit")
logger.warning("Retry attempt \(attempt)")
logger.error("Request failed: \(error.localizedDescription)")
logger.critical("Fatal error")
logger.fault("Unrecoverable error")

// Privacy levels
logger.debug("Token: \(token, privacy: .private)")    // redacted in logs
logger.debug("User ID: \(userId, privacy: .public)")  // visible
logger.debug("Sensitive: \(data, privacy: .sensitive)") // redacted + marked sensitive

// Legacy os_log
os_log("User %@ logged in", log: OSLog(subsystem: "com.example.app", category: "Auth"), type: .info, userId)
os_log_error("Failed: %{public}@", error.localizedDescription)
```

- **Privacy**: `.public` (visible), `.private` (redacted), `.sensitive` (redacted + sensitive). Default is `.private` for strings.
- **Levels**: `.debug` (verbose, off by default), `.info` (general), `.notice` (default, important), `.warning` (problems), `.error` (errors), `.fault` (unrecoverable), `.critical` (system-level).
- **Subsystem**: reverse-DNS bundle ID; **category**: feature area.
- **Stream**: `log stream --predicate 'subsystem == "com.example.app"' --level debug`.
- **History**: `log show --last 1h --predicate 'subsystem == "com.example.app"'`.
- **Signposts**: `os_signpost` for Instruments intervals.
- **Never log**: tokens, passwords, PII, full request/response bodies with sensitive data.

## 6. Console.app & log commands

```bash
# Stream logs
log stream --predicate 'subsystem == "com.example.app"' --level debug
log stream --predicate 'process == "MyApp"' --style compact
log stream --predicate 'eventMessage CONTAINS "error"' --level error

# Show historical logs
log show --last 1h --predicate 'subsystem == "com.example.app"'
log show --start "2026-01-01 00:00:00" --end "2026-01-01 01:00:00" --predicate 'subsystem == "com.example.app"'
log show --predicate 'eventMessage CONTAINS "crash"' --last 24h

# Device logs
xcrun devicectl device console --device <udid>

# Simulator logs
xcrun simctl spawn booted log stream --level debug
```

- `log` command — unified logging system; `--predicate` for filtering (subsystem, category, process, eventMessage, messageType).
- `log show` — historical logs from log archive.
- `log stream` — live log stream.
- `log collect` — collect full log archive.
- `log erase` — clear log archive (requires sudo).

## 7. Crash reports & symbolication

### Crash report types
- `.ips` — JSON crash reports in `~/Library/Logs/DiagnosticReports/`.
- `.crash` — legacy format.
- `.hang` — hang reports.
- `.spin` — CPU spin reports.
- `.apptermination` — termination reports.

### Symbolication
```bash
# Manual symbolication
xcrun symbolicatecrash crash.ips App.app.dSYM > symbolicated.crash

# Or use atos
atos -o App.app.dSYM/Contents/Resources/DWARF/App -l 0x100000000 0x100001234

# Or lldb
(lldb) image lookup -a 0x100001234 -s App.app.dSYM
```

- **dSYM** — debug symbols generated in Release builds (`DEBUG_INFORMATION_FORMAT = dwarf-with-dsym`).
- Upload dSYMs to App Store Connect / crash reporting service for symbolication.
- ` atos` — address-to-symbol lookup.
- `atos -o binary -l loadAddress address` — symbolicate address.

### MetricKit
```swift
import MetricKit

class MetricsManager: NSObject, MXMetricManagerSubscriber {
  func didReceive(_ payloads: [MXMetricPayload]) {
    for payload in payloads {
      // payload.jsonRepresentation() — metrics data
      // Daily metrics: CPU, memory, GPU, battery, disk I/O
    }
  }
  func didReceive(_ payloads: [MXDiagnosticPayload]) {
    for payload in payloads {
      // payload.crashDiagnostics — crash reports
      // payload.hangDiagnostics — hang reports
      // payload.cpuExceptionDiagnostics — CPU exceptions
      // payload.diskWriteExceptionDiagnostics — disk write exceptions
      // payload.appExitDiagnostics — app exits
    }
  }
}

// Register subscriber
let manager = MXMetricManager.shared
manager.add(MetricsManager())
```

- `MXMetricPayload` — daily performance metrics (CPU, memory, GPU, battery, disk).
- `MXDiagnosticPayload` — crash/hang/exception diagnostics.
- Delivered once per day; requires app to have been running.
- `MXMetricManagerSubscriber` — implement `didReceive` for both payload types.

## 8. Localization

### String Catalogs (.xcstrings)
```json
{
  "sourceLanguage" : "en",
  "strings" : {
    "Hello %@" : {
      "localizations" : {
        "en" : { "stringUnit" : { "state" : "translated", "value" : "Hello %@" } },
        "es" : { "stringUnit" : { "state" : "translated", "value" : "Hola %@" } }
      }
    },
    "^[%lld item](inflect: true)" : {
      "localizations" : {
        "en" : {
          "variations" : {
            "plural" : {
              "one" : { "stringUnit" : { "state" : "translated", "value" : "%lld item" } },
              "other" : { "stringUnit" : { "state" : "translated", "value" : "%lld items" } }
            }
          }
        }
      }
    }
  },
  "version" : "1.0"
}
```

- `.xcstrings` — JSON-based string catalog; keys = source strings; values per locale.
- **Automatic extraction**: Xcode scans `Text("...")`, `String(localized:)`, `LocalizedStringKey` for keys.
- **Plural variations**: `^[%lld item](inflect: true)` syntax for plural rules.
- **Interpolation**: `%@`, `%lld`, `%f`, `%@` for objects; `\(variable)` in SwiftUI `Text`.
- **Comments**: `/* comment */` for translator context.
- **State**: `translated`, `needs_review`, `new`.

### Export/Import
```bash
# Export for localization
xcodebuild -exportLocalizations -project MyApp.xcodeproj -localizationPath ./Localizations

# Import translations
xcodebuild -importLocalizations -project MyApp.xcodeproj -localizationPath ./Localizations
```

- Xcode: Editor > Export For Localization / Import Localizations.
- `.xcloc` bundles for translation vendors.

### Best practices
- Use `Text("key")` in SwiftUI — automatic localization.
- Use `String(localized: "key", defaultValue: "...", comment: "...")` for non-View code.
- Use `LocalizedStringResource` for intent titles.
- Avoid concatenating localized strings — use format strings with placeholders.
- Test with pseudo-localization scheme (Product > Scheme > Edit > Run > Options > App Language > Double Length Pseudolanguage).
- Test RTL: pseudolanguage or Arabic/Hebrew locale.
- `String(localized:)` supports `table:` for multiple string tables.

## 9. Asset catalogs

### AppIcon
- `Assets.xcassets/AppIcon.appiconset` — all required sizes (iPhone, iPad, Mac, Watch, Vision).
- Icon Composer (`.icon` format, iOS 18+) — layered icons with Liquid Glass effects.
- `ASSETCATALOG_COMPILER_APPICON_NAME` — build setting for icon name.
- Dark/tinted variants: `Appearances` → `luminosity: dark`, `luminosity: tinted`.

### Colors
- `Colors.colorset` — Any/Dark/High Contrast appearances.
- Named colors: `Color("Primary")` in SwiftUI, `UIColor(named: "Primary")` in UIKit.
- Dynamic colors adapt automatically.

### Images
- `Images.imageset` — universal/any scale (1x/2x/3x), dark variants, vector PDFs.
- `preserves-vector-representation` for scalable PDFs.
- `symbol` imagesets — custom SF Symbols.

### Data sets
- `DataSets.dataset` — ML models, configuration files.
- `MLModel` compilation on build.

### SF Symbols
- `Symbol Set` — custom symbol sets.
- `Symbols` — SF Symbols app for browsing/configuring.
- `symbolVariants` — `.fill`, `.circle`, `.square`, `.rectangle`.

## 10. Performance & build optimization

### Build settings for release
- `SWIFT_OPTIMIZATION_LEVEL = -O` (or `-Osize` for size).
- `SWIFT_COMPILATION_MODE = wholemodule` (WMO — faster, smaller).
- `DEBUG_INFORMATION_FORMAT = dwarf-with-dsym`.
- `SWIFT_ACTIVE_COMPILATION_CONDITIONS` — remove `DEBUG`.
- `GCC_OPTIMIZATION_LEVEL = s` (size) or `fast` (speed).
- `DEAD_CODE_STRIPPING = YES`.
- `STRIP_INSTALLED_PRODUCT = YES`.
- `CLANG_ENABLE_MODULES = YES`.
- `ENABLE_BITCODE` — **REMOVED** in modern Xcode (deprecated since Xcode 14).

### Launch time
- `DYLD_PRINT_STATISTICS = 1` — launch time breakdown.
- `DYLD_PRINT_APIS = 1` — dyld API calls.
- `DYLD_PRINT_BINDINGS = 1` — symbol bindings.
- `os_signpost` around launch phases.
- `XCTApplicationLaunchMetric` for measurement.
- Defer non-critical init; avoid heavy static initializers; lazy-load resources.

### App size
- `STRIP_SWIFT_SYMBOLS = YES`.
- `SWIFT_EMIT_LOC_STRINGS = NO` (reduces binary size).
- Remove unused resources; use asset catalogs for compression.
- On-demand resources (ODR) for large assets — tags in asset catalog.
- App thinning: App Store slices per device automatically.

### Runtime performance
- Instruments: Time Profiler, Allocations, Leaks, Core Animation, Animation Hitches, Hangs.
- `os_signpost` for custom metrics.
- `XCTMetric` for XCTest performance tests.
- `CADisplayLink` for frame-synced work.
- `DispatchQueue` QoS for priority.
- `Task` priorities for concurrency.

## 11. CI/CD with Xcode

### xcodebuild in CI
```bash
# Build
xcodebuild -scheme MyApp -destination 'generic/platform=iOS' -configuration Release \
  -archivePath MyApp.xcarchive archive \
  -allowProvisioningUpdates \
  CODE_SIGN_STYLE=Automatic \
  DEVELOPMENT_TEAM=TEAMID

# Export
xcodebuild -exportArchive -archivePath MyApp.xcarchive \
  -exportOptionsPlist ExportOptions.plist \
  -exportPath ./export

# Test
xcodebuild test -scheme MyApp -destination 'platform=iOS Simulator,name=iPhone 17' \
  -resultBundlePath TestResults.xcresult \
  -parallel-testing-enabled YES \
  -maximum-parallel-testing-workers 4

# Upload to App Store Connect (via altool or Transporter)
xcrun altool --upload-app -f MyApp.ipa -t ios --apiKey KEY --apiIssuer ISSUER
# or use Transporter app / iTMSTransporter
```

### ExportOptions.plist
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>method</key>
  <string>app-store-connect</string>  <!-- or ad-hoc, enterprise, development, mac-application, developer-id -->
  <key>teamID</key>
  <string>TEAMID</string>
  <key>uploadSymbols</key>
  <true/>
  <key>uploadBitcode</key>
  <false/>  <!-- bitcode removed -->
  <key>signingStyle</key>
  <string>automatic</string>  <!-- or manual -->
  <key>provisioningProfiles</key>
  <dict>
    <key>com.example.app</key>
    <string>App Store Profile</string>
  </dict>
</dict>
</plist>
```

### Export methods
- `app-store-connect` — App Store / TestFlight.
- `ad-hoc` — registered devices.
- `enterprise` — enterprise distribution.
- `development` — development devices.
- `mac-application` — Mac App Store.
- `developer-id` — Mac direct distribution.

### Xcode Cloud
- Connect git repo → automatic builds on push/PR.
- `ci_scripts/` — custom scripts:
  - `ci_post_clone.sh` — after clone (install deps, setup).
  - `ci_pre_xcodebuild.sh` — before build.
  - `ci_post_xcodebuild.sh` — after build (upload artifacts, notify).
- Environment: `CI`, `CI_BUILD_ID`, `CI_COMMIT`, `CI_BRANCH`, `CI_TAG`, `CI_WORKSPACE`.
- Test results in App Store Connect.

### Fastlane
```ruby
# Fastfile
lane :build do
  gym(scheme: "MyApp", workspace: "MyApp.xcworkspace", export_method: "app-store")
end

lane :test do
  scan(scheme: "MyApp", device: "iPhone 17")
end

lane :beta do
  build
  pilot(skip_waiting_for_build_processing: true)
end
```

- `gym` — build IPA; `scan` — run tests; `pilot` — TestFlight upload; `deliver` — App Store metadata; `match` — cert/profile management; `sigh` — provisioning profiles; `cert` — certificates.

## 12. Common Xcode issues → fixes

| Issue | Fix |
|---|---|
| "No such module" | check target membership, `SWIFT_INCLUDE_PATHS`, module map |
| "Cannot find 'X' in scope" | missing import, typo, wrong target |
| Build succeeds but app crashes on launch | check Info.plist, entitlements, signing |
| "Provisioning profile doesn't include device" | register device, update profile, `-allowProvisioningDeviceRegistration` |
| "No profiles for bundle ID" | check bundle ID, team, capabilities |
| Archive fails with signing error | check certs, profiles, `CODE_SIGN_IDENTITY`, `PROVISIONING_PROFILE_SPECIFIER` |
| Tests not discovered | check test target membership, scheme test action |
| Slow builds | enable WMO, reduce dependencies, use binary cache (Tuist), incremental builds |
| DerivedData issues | `xcodebuild clean`, delete `~/Library/Developer/Xcode/DerivedData` |
| Simulator won't boot | `xcrun simctl erase all`, restart Simulator |
| Device not recognized | check cable, trust, `xcrun devicectl list devices` |
| Code coverage missing | enable `CODE_COVERAGE_ENABLED` in scheme or `-enableCodeCoverage` |
| Swift 6 concurrency errors | fix Sendable/actor issues, use `-strict-concurrency=complete` for early detection |
| Previews not working | check deployment target, `#Preview` syntax, canvas refresh |
| Asset catalog warnings | check image sizes, PDF vector representation, dark variants |
| Localization missing | check `.xcstrings` file, target membership, string extraction |
