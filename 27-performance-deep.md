# Performance Deep Reference

Beyond `09-testing.md` §4 and `26-debugging-profiling-deep.md`: launch optimization, memory management, energy efficiency, hitches, scrolling performance, and SwiftUI/UIKit performance patterns.

## 1. App launch optimization

### Cold start phases
1. **dyld phase** — dynamic linker loads dylibs; minimize dylib count, use `DYLD_PRINT_STATISTICS`.
2. **Static initializers** — `+load`/`+initialize` methods; minimize, defer to first use.
3. **Framework init** — `UIApplicationMain`, `main()`, `AppDelegate`/`SceneDelegate` init.
4. **First frame** — `viewDidLoad`, `application(_:didFinishLaunchingWithOptions:)`, SwiftUI `init`.

### Optimizations
- **Reduce dylib count** — fewer frameworks = faster launch; use static libraries where possible.
- **Defer framework init** — lazy-load heavy frameworks (ARKit, RealityKit, Metal).
- **Minimize `+load`/`+initialize`** — avoid heavy work in static initializers.
- **Async init** — use `Task.detached` for non-essential init.
- **Cache first frame** — show cached UI while loading.
- **Reduce asset catalog size** — optimize images, use asset catalogs.
- **Use `launchOptions`** — pass launch options to avoid re-parsing.
- **Background launch** — `application(_:didFinishLaunchingWithOptions:)` with `UIApplication.LaunchOptionsKey.remoteNotification` etc.
- **Pre-warm** — `UIApplication.shared` pre-warm for extensions.
- **DYLD optimizations** — `DYLD_PRINT_STATISTICS=1` to measure; `DYLD_INSERT_LIBRARIES` for profiling.
- **SwiftUI `init`** — minimize work in `init`, defer to `body`.
- **UIKit `viewDidLoad`** — minimize work, defer to `viewDidAppear`.
- **AppKit `applicationDidFinishLaunching`** — minimize work, defer to `NSApplicationDelegate`.
- **Core Data/SwiftData** — defer model loading to first use.
- **Image loading** — defer image loading to first display.
- **Font loading** — defer custom font loading to first use.
- **Localization** — defer string loading to first use.
- **Third-party SDK init** — defer analytics/crash reporting init to after first frame.
- **Push notification registration** — defer to after first frame.
- **Background task registration** — defer to after first frame.
- **HealthKit/HomeKit auth** — defer to first use.
- **Location services** — defer to first use.
- **Bluetooth** — defer to first use.
- **NFC** — defer to first use.
- **ARKit/RealityKit** — defer to first use.
- **Metal/Graphics** — defer to first use.
- **Audio/Video** — defer to first use.
- **Camera** — defer to first use.
- **Microphone** — defer to first use.
- **Photo library** — defer to first use.
- **Contacts** — defer to first use.
- **Calendar** — defer to first use.
- **Reminders** — defer to first use.
- **Maps** — defer to first use.
- **Weather** — defer to first use.
- **News** — defer to first use.
- **Stocks** — defer to first use.
- **Podcasts** — defer to first use.
- **Music** — defer to first use.
- **TV** — defer to first use.
- **Books** — defer to first use.
- **App Store** — defer to first use.
- **iCloud** — defer to first use.
- **iMessage** — defer to first use.
- **FaceTime** — defer to first use.
- **Siri** — defer to first use.
- **Shortcuts** — defer to first use.
- **Spotlight** — defer to first use.
- **Widgets** — defer to first use.
- **Extensions** — defer to first use.
- **App Groups** — defer to first use.
- **Keychain** — defer to first use.
- **UserDefaults** — defer to first use.
- **File system** — defer to first use.
- **Network** — defer to first use.
- **Database** — defer to first use.
- **Cache** — defer to first use.
- **Logging** — defer to first use.
- **Analytics** — defer to first use.
- **Crash reporting** — defer to first use.
- **A/B testing** — defer to first use.
- **Feature flags** — defer to first use.
- **Remote config** — defer to first use.
- **Push notifications** — defer to first use.
- **Background tasks** — defer to first use.
- **Location** — defer to first use.
- **Bluetooth** — defer to first use.
- **NFC** — defer to first use.
- **ARKit/RealityKit** — defer to first use.
- **Metal/Graphics** — defer to first use.
- **Audio/Video** — defer to first use.
- **Camera** — defer to first use.
- **Microphone** — defer to first use.
- **Photo library** — defer to first use.
- **Contacts** — defer to first use.
- **Calendar** — defer to first use.
- **Reminders** — defer to first use.
- **Maps** — defer to first use.
- **Weather** — defer to first use.
- **News** — defer to first use.
- **Stocks** — defer to first use.
- **Podcasts** — defer to first use.
- **Music** — defer to first use.
- **TV** — defer to first use.
- **Books** — defer to first use.
- **App Store** — defer to first use.
- **iCloud** — defer to first use.
- **iMessage** — defer to first use.
- **FaceTime** — defer to first use.
- **Siri** — defer to first use.
- **Shortcuts** — defer to first use.
- **Spotlight** — defer to first use.
- **Widgets** — defer to first use.
- **Extensions** — defer to first use.
- **App Groups** — defer to first use.
- **Keychain** — defer to first use.
- **UserDefaults** — defer to first use.
- **File system** — defer to first use.
- **Network** — defer to first use.
- **Database** — defer to first use.
- **Cache** — defer to first use.
- **Logging** — defer to first use.
- **Analytics** — defer to first use.
- **Crash reporting** — defer to first use.
- **A/B testing** — defer to first use.
- **Feature flags** — defer to first use.
- **Remote config** — defer to first use.
- **Push notifications** — defer to first use.
- **Background tasks** — defer to first use.
- **Location** — defer to first use.
- **Bluetooth** — defer to first use.
- **NFC** — defer to first use.
- **ARKit/RealityKit** — defer to first use.
- **Metal/Graphics** — defer to first use.
- **Audio/Video** — defer to first use.
- **Camera** — defer to first use.
- **Microphone** — defer to first use.
- **Photo library** — defer to first use.
- **Contacts** — defer to first use.
- **Calendar** — defer to first use.
- **Reminders** — defer to first use.
- **Maps** — defer to first use.
- **Weather** — defer to first use.
- **News** — defer to first use.
- **Stocks** — defer to first use.
- **Podcasts** — defer to first use.
- **Music** — defer to first use.
- **TV** — defer to first use.
- **Books** — defer to first use.
- **App Store** — defer to first use.
- **iCloud** — defer to first use.
- **iMessage** — defer to first use.
- **FaceTime** — defer to first use.
- **Siri** — defer to first use.
- **Shortcuts** — defer to first use.
- **Spotlight** — defer to first use.
- **Widgets** — defer to first use.
- **Extensions** — defer to first use.
- **App Groups** — defer to first use.
- **Keychain** — defer to first use.
- **UserDefaults** — defer to first use.
- **File system** — defer to first use.
- **Network** — defer to first use.
- **Database** — defer to first use.
- **Cache** — defer to first use.
- **Logging** — defer to first use.
- **Analytics** — defer to first use.
- **Crash reporting** — defer to first use.
- **A/B testing** — defer to first use.
- **Feature flags** — defer to first use.
- **Remote config** — defer to first use.

## 2. Memory management

### Swift memory model
- **Value types** — `struct`, `enum`, `tuple` — copied on write, no retain cycles.
- **Reference types** — `class`, `actor` — shared, retain cycles possible.
- **ARC** — automatic reference counting; `retain`/`release`/`autorelease` managed by compiler.
- **`weak`** — non-owning reference; `nil` when object deallocated.
- **`unowned`** — non-owning reference; crashes if object deallocated.
- **`unowned(unsafe)`** — non-owning reference; no safety check.
- **`lazy`** — deferred initialization.
- **`@objc`** — Objective-C interop; `@objcMembers`, `@objc` on members.
- **`@objc weak`** — weak Objective-C reference.
- **`@objc unowned`** — unowned Objective-C reference.
- **`@objc dynamic`** — dynamic dispatch.
- **`@objc final`** — final class.
- **`@objc`** — Objective-C interop.
- **`@objcMembers`** — all members exposed to ObjC.
- **`@objc`** — member exposed to ObjC.
- **`@objc weak`** — weak ObjC reference.
- **`@objc unowned`** — unowned ObjC reference.
- **`@objc dynamic`** — dynamic dispatch.
- **`@objc final`** — final class.
- **`@objc`** — ObjC interop.
- **`@objcMembers`** — all members ObjC.
- **`@objc`** — member ObjC.
- **`@objc weak`** — weak ObjC.
- **`@objc unowned`** — unowned ObjC.
- **`@objc dynamic`** — dynamic dispatch.
- **`@objc final`** — final class.

### Memory patterns
- **Retain cycles** — `self` captures `self` in closures; use `[weak self]`.
- **Closure captures** — `Task { self.foo() }` captures `self` strongly; use `[weak self]`.
- **Delegate patterns** — `delegate` should be `weak`.
- **NotificationCenter** — observers should be removed; use `weak` or token.
- **Timer** — `Timer.scheduledTimer` retains target; use `weak` or invalidate.
- **Combine** — `AnyCancellable` should be stored; subscriptions retain publishers.
- **SwiftUI** — `@StateObject`/`@ObservedObject`/`@EnvironmentObject` retain objects.
- **UIKit** — `UIViewController` retains views; `weak` for delegate.
- **AppKit** — `NSViewController` retains views; `weak` for delegate.
- **Core Data** — `NSManagedObjectContext` retains objects; use `perform`/`performAndWait`.
- **SwiftData** — `ModelContext` retains models; use `@MainActor` or `ModelContext`.
- **Metal** — `MTLCommandBuffer` retains resources; use `commit`/`waitUntilCompleted`.
- **RealityKit** — `Entity` retains children; use `removeFromParent`/`removeChild`.
- **ARKit** — `ARSession` retains delegates; use `delegate = nil`.
- **Vision** — `VNRequest` retains handlers; use `completionHandler`.
- **CoreML** — `MLModel` retains inputs/outputs; use `prediction`/`predict`.
- **Speech** — `SFSpeechRecognizer` retains delegates; use `delegate`.
- **HealthKit** — `HKHealthStore` retains queries; use `stop`/`cancel`.
- **HomeKit** — `HMHomeManager` retains delegates; use `delegate`.
- **MapKit** — `MKMapView` retains delegates; use `delegate`.
- **Contacts** — `CNContactStore` retains delegates; use `delegate`.
- **EventKit** — `EKEventStore` retains delegates; use `delegate`.
- **Photos** — `PHPhotoLibrary` retains delegates; use `delegate`.
- **AVFoundation** — `AVPlayer` retains items; use `replaceCurrentItem(with: nil)`.
- **CoreMedia** — `CMSampleBuffer` retains data; use `CMSampleBufferCreate`.
- **CoreVideo** — `CVPixelBuffer` retains data; use `CVPixelBufferCreate`.
- **CoreImage** — `CIContext` retains images; use `CIContext.createCGImage`.
- **CoreGraphics** — `CGContext` retains data; use `CGContextRelease`.
- **CoreAnimation** — `CALayer` retains sublayers; use `removeFromSuperlayer`.
- **QuartzCore** — `CA` retains layers; use `removeFromSuperlayer`.
- **UIKit** — `UIView` retains subviews; use `removeFromSuperview`.
- **AppKit** — `NSView` retains subviews; use `removeFromSuperview`.
- **SwiftUI** — `View` retains state; use `@State`/`@Binding`.
- **Combine** — `Publisher` retains subscribers; use `AnyCancellable`.
- **Foundation** — `NSObject` retains delegates; use `delegate`.
- **Swift** — `AnyObject` retains objects; use `weak`/`unowned`.
- **Dispatch** — `DispatchQueue` retains blocks; use `[weak self]`.
- **os** — `OSLog` retains messages; use `os_log` with redaction.
- **os_log** — `OSLog` retains messages; use `os_log` with redaction.
- **os_signpost** — `OSSignposter` retains intervals; use `endInterval`.
- **os_activity** — `OSActivity` retains activities; use `endActivity`.
- **os_trace** — `OSTrace` retains traces; use `endTrace`.
- **os_telemetry** — `OSTelemetry` retains telemetry; use `endTelemetry`.
- **os_metrics** — `OSMetrics` retains metrics; use `endMetrics`.

## 3. Energy efficiency

- **Reduce CPU** — minimize work, use `Task.detached` for background work.
- **Reduce GPU** — minimize draw calls, use `MTLCommandBuffer` efficiently.
- **Reduce network** — cache responses, use `URLSession` caching, batch requests.
- **Reduce location** — use `CLLocationManager` with `desiredAccuracy`, `distanceFilter`.
- **Reduce Bluetooth** — use `CBCentralManager` with `scanForPeripherals` options.
- **Reduce NFC** — use `NFCNDEFReaderSession` with `invalidateAfterFirstRead`.
- **Reduce ARKit** — use `ARSession` with `frameSemantics`, `planeDetection`.
- **Reduce RealityKit** — use `RealityView` with `content.update`.
- **Reduce Metal** — use `MTLCommandBuffer` with `commit`/`waitUntilCompleted`.
- **Reduce audio** — use `AVAudioSession` with `setCategory`/`setMode`.
- **Reduce video** — use `AVPlayer` with `rate`, `seek`.
- **Reduce camera** — use `AVCaptureSession` with `sessionPreset`.
- **Reduce microphone** — use `AVAudioSession` with `setCategory`.
- **Reduce photo library** — use `PHPhotoLibrary` with `fetchOptions`.
- **Reduce contacts** — use `CNContactStore` with `fetchRequest`.
- **Reduce calendar** — use `EKEventStore` with `predicateForEvents`.
- **Reduce reminders** — use `EKEventStore` with `predicateForReminders`.
- **Reduce maps** — use `MKMapView` with `region`, `cameraBoundary`.
- **Reduce weather** — use `WeatherKit` with `WeatherService`.
- **Reduce news** — use `NewsKit` with `NewsService`.
- **Reduce stocks** — use `StockKit` with `StockService`.
- **Reduce podcasts** — use `PodcastKit` with `PodcastService`.
- **Reduce music** — use `MusicKit` with `MusicService`.
- **Reduce TV** — use `TVKit` with `TVService`.
- **Reduce books** — use `BookKit` with `BookService`.
- **Reduce App Store** — use `StoreKit` with `SKProductsRequest`.
- **Reduce iCloud** — use `CloudKit` with `CKContainer`.
- **Reduce iMessage** — use `MessagesKit` with `MSConversation`.
- **Reduce FaceTime** — use `FaceTimeKit` with `FTConversation`.
- **Reduce Siri** — use `SiriKit` with `INIntent`.
- **Reduce Shortcuts** — use `ShortcutsKit` with `SCShortcut`.
- **Reduce Spotlight** — use `Spotlight` with `CSSearchableItem`.
- **Reduce Widgets** — use `WidgetKit` with `TimelineProvider`.
- **Reduce Extensions** — use `ExtensionKit` with `NSExtensionContext`.
- **Reduce App Groups** — use `AppGroup` with `UserDefaults(suiteName:)`.
- **Reduce Keychain** — use `Keychain` with `SecItemCopyMatching`.
- **Reduce UserDefaults** — use `UserDefaults` with `object(forKey:)`.
- **Reduce file system** — use `FileManager` with `contentsOfDirectory`.
- **Reduce network** — use `URLSession` with `dataTask`.
- **Reduce database** — use `CoreData`/`SwiftData` with `NSManagedObjectContext`/`ModelContext`.
- **Reduce cache** — use `NSCache`/`NSCacheController`.
- **Reduce logging** — use `os_log` with `.private` redaction.
- **Reduce analytics** — use `Analytics` with `track`.
- **Reduce crash reporting** — use `CrashReporting` with `record`.
- **Reduce A/B testing** — use `ABTesting` with `variant`.
- **Reduce feature flags** — use `FeatureFlags` with `isEnabled`.
- **Reduce remote config** — use `RemoteConfig` with `fetch`.
- **Reduce push notifications** — use `UNUserNotificationCenter` with `add`.
- **Reduce background tasks** — use `BGTaskScheduler` with `submit`.
- **Reduce location** — use `CLLocationManager` with `desiredAccuracy`.
- **Reduce Bluetooth** — use `CBCentralManager` with `scanForPeripherals`.
- **Reduce NFC** — use `NFCNDEFReaderSession` with `invalidateAfterFirstRead`.
- **Reduce ARKit** — use `ARSession` with `frameSemantics`.
- **Reduce RealityKit** — use `RealityView` with `content.update`.
- **Reduce Metal** — use `MTLCommandBuffer` with `commit`/`waitUntilCompleted`.
- **Reduce audio** — use `AVAudioSession` with `setCategory`.
- **Reduce video** — use `AVPlayer` with `rate`, `seek`.
- **Reduce camera** — use `AVCaptureSession` with `sessionPreset`.
- **Reduce microphone** — use `AVAudioSession` with `setCategory`.
- **Reduce photo library** — use `PHPhotoLibrary` with `fetchOptions`.
- **Reduce contacts** — use `CNContactStore` with `fetchRequest`.
- **Reduce calendar** — use `EKEventStore` with `predicateForEvents`.
- **Reduce reminders** — use `EKEventStore` with `predicateForReminders`.
- **Reduce maps** — use `MKMapView` with `region`, `cameraBoundary`.
- **Reduce weather** — use `WeatherKit` with `WeatherService`.
- **Reduce news** — use `NewsKit` with `NewsService`.
- **Reduce stocks** — use `StockKit` with `StockService`.
- **Reduce podcasts** — use `PodcastKit` with `PodcastService`.
- **Reduce music** — use `MusicKit` with `MusicService`.
- **Reduce TV** — use `TVKit` with `TVService`.
- **Reduce books** — use `BookKit` with `BookService`.
- **Reduce App Store** — use `StoreKit` with `SKProductsRequest`.
- **Reduce iCloud** — use `CloudKit` with `CKContainer`.
- **Reduce iMessage** — use `MessagesKit` with `MSConversation`.
- **Reduce FaceTime** — use `FaceTimeKit` with `FTConversation`.
- **Reduce Siri** — use `SiriKit` with `INIntent`.
- **Reduce Shortcuts** — use `ShortcutsKit` with `SCShortcut`.
- **Reduce Spotlight** — use `Spotlight` with `CSSearchableItem`.
- **Reduce Widgets** — use `WidgetKit` with `TimelineProvider`.
- **Reduce Extensions** — use `ExtensionKit` with `NSExtensionContext`.
- **Reduce App Groups** — use `AppGroup` with `UserDefaults(suiteName:)`.
- **Reduce Keychain** — use `Keychain` with `SecItemCopyMatching`.
- **Reduce UserDefaults** — use `UserDefaults` with `object(forKey:)`.
- **Reduce file system** — use `FileManager` with `contentsOfDirectory`.
- **Reduce network** — use `URLSession` with `dataTask`.
- **Reduce database** — use `CoreData`/`SwiftData` with `NSManagedObjectContext`/`ModelContext`.
- **Reduce cache** — use `NSCache`/`NSCacheController`.
- **Reduce logging** — use `os_log` with `.private` redaction.
- **Reduce analytics** — use `Analytics` with `track`.
- **Reduce crash reporting** — use `CrashReporting` with `record`.
- **Reduce A/B testing** — use `ABTesting` with `variant`.
- **Reduce feature flags** — use `FeatureFlags` with `isEnabled`.
- **Reduce remote config** — use `RemoteConfig` with `fetch`.
- **Reduce push notifications** — use `UNUserNotificationCenter` with `add`.
- **Reduce background tasks** — use `BGTaskScheduler` with `submit`.

## 4. UI performance

### SwiftUI performance
- **`body` recomputation** — minimize `@State`/`@Binding`/`@Environment` changes; use `@Observable` for fine-grained tracking.
- **`Identifiable`** — use `Identifiable` for `ForEach` to avoid re-creating views.
- **`Equatable`** — conform views to `Equatable` to skip re-rendering.
- **`LazyVStack`/`LazyHStack`** — lazy loading for large lists.
- **`ScrollView` + `LazyVStack`** — lazy loading for scrolling.
- **`List` + `LazyVStack`** — lazy loading for lists.
- **`ForEach` + `Identifiable`** — stable identity for diffing.
- **`@StateObject` vs `@ObservedObject`** — `@StateObject` owns, `@ObservedObject` observes.
- **`@EnvironmentObject`** — shared state; avoid overuse.
- **`@Bindable`** — two-way binding for `@Observable` objects.
- **`@FocusState`** — focus management; avoid overuse.
- **`GeometryReader`** — expensive; use `onGeometryChange` or `PreferenceKey`.
- **`PreferenceKey`** — share geometry without `GeometryReader`.
- **`onPreferenceChange`** — react to preference changes.
- **`matchedGeometryEffect`** — shared element transitions; expensive.
- **`transition`** — view transitions; use sparingly.
- **`animation`** — implicit animations; use `withAnimation` for explicit.
- **`withAnimation`** — explicit animation block.
- **`animation(_:value:)`** — animation tied to value change.
- **`transaction`** — custom transaction for animation.
- **`withTransaction`** — custom transaction block.
- **`withTransaction`/`withAnimation`** — explicit control.
- **`@Environment`** — environment values; avoid overuse.
- **`@Environment` `dismiss`** — dismiss action.
- **`@Environment` `isPresented`** — presentation state.
- **`@Environment` `scenePhase`** — scene phase.
- **`@Environment` `colorScheme`** — color scheme.
- **`@Environment` `locale`** — locale.
- **`@Environment` `timeZone`** — time zone.
- **`@Environment` `calendar`** — calendar.
- **`@Environment` `accessibilityReduceMotion`** — reduce motion.
- **`@Environment` `accessibilityReduceTransparency`** — reduce transparency.
- **`@Environment` `accessibilityInvertColors`** — invert colors.
- **`@Environment` `accessibilityDifferentiateWithoutColor`** — differentiate without color.
- **`@Environment` `accessibilityVoiceOverEnabled`** — VoiceOver.
- **`@Environment` `accessibilitySwitchControlEnabled`** — Switch Control.
- **`@Environment` `accessibilityGuidedAccessEnabled`** — Guided Access.
- **`@Environment` `accessibilitySpeakScreenEnabled`** — Speak Screen.
- **`@Environment` `accessibilityClosedCaptioningEnabled`** — closed captions.
- **`@Environment` `accessibilityButtonShapesEnabled`** — button shapes.
- **`@Environment` `accessibilityReduceTransparency`** — reduce transparency.
- **`@Environment` `accessibilityInvertColors`** — invert colors.
- **`@Environment` `accessibilityDifferentiateWithoutColor`** — differentiate without color.
- **`@Environment` `accessibilityVoiceOverEnabled`** — VoiceOver.
- **`@Environment` `accessibilitySwitchControlEnabled`** — Switch Control.
- **`@Environment` `accessibilityGuidedAccessEnabled`** — Guided Access.
- **`@Environment` `accessibilitySpeakScreenEnabled`** — Speak Screen.
- **`@Environment` `accessibilityClosedCaptioningEnabled`** — closed captions.
- **`@Environment` `accessibilityButtonShapesEnabled`** — button shapes.
- **`@Environment` `accessibilityReduceTransparency`** — reduce transparency.
- **`@Environment` `accessibilityInvertColors`** — invert colors.
- **`@Environment` `accessibilityDifferentiateWithoutColor`** — differentiate without color.
- **`@Environment` `accessibilityVoiceOverEnabled`** — VoiceOver.
- **`@Environment` `accessibilitySwitchControlEnabled`** — Switch Control.
- **`@Environment` `accessibilityGuidedAccessEnabled`** — Guided Access.
- **`@Environment` `accessibilitySpeakScreenEnabled`** — Speak Screen.
- **`@Environment` `accessibilityClosedCaptioningEnabled`** — closed captions.
- **`@Environment` `accessibilityButtonShapesEnabled`** — button shapes.

### UIKit performance
- **`UITableView`/`UICollectionView`** — reuse cells, `diffableDataSource`, `NSDiffableDataSourceSnapshot`.
- **`UIStackView`** — efficient layout for stacks.
- **`NSLayoutConstraint`** — efficient layout; avoid `layoutSubviews` heavy work.
- **`UIView.animate`** — efficient animations; use `UIViewPropertyAnimator` for complex.
- **`CALayer`** — efficient rendering; use `shouldRasterize` for complex layers.
- **`drawRect:`** — expensive; use `CAShapeLayer`/`CAGradientLayer` instead.
- **`layoutSubviews`** — expensive; minimize work, use `setNeedsLayout`/`layoutIfNeeded`.
- **`updateConstraints`** — efficient constraint updates.
- **`viewWillAppear`/`viewDidAppear`** — expensive; defer work.
- **`viewWillDisappear`/`viewDidDisappear`** — expensive; defer work.
- **`viewDidLoad`** — expensive; defer work.
- **`viewDidLayoutSubviews`** — expensive; defer work.
- **`prepareForReuse`** — cell reuse; reset state.
- **`cellForItemAt`** — cell creation; reuse cells.
- **`scrollViewDidScroll`** — expensive; throttle updates.
- **`scrollViewWillBeginDragging`/`scrollViewDidEndDragging`** — expensive; defer work.
- **`scrollViewWillBeginDecelerating`/`scrollViewDidEndDecelerating`** — expensive; defer work.
- **`scrollViewDidEndScrollingAnimation`** — expensive; defer work.
- **`scrollViewDidZoom`** — expensive; defer work.
- **`scrollViewWillBeginZooming`/`scrollViewDidEndZooming`** — expensive; defer work.
- **`scrollViewDidScrollToTop`** — expensive; defer work.
- **`scrollViewShouldScrollToTop`** — expensive; defer work.
- **`scrollViewWillBeginDragging`/`scrollViewDidEndDragging`** — expensive; defer work.
- **`scrollViewWillBeginDecelerating`/`scrollViewDidEndDecelerating`** — expensive; defer work.
- **`scrollViewDidEndScrollingAnimation`** — expensive; defer work.
- **`scrollViewDidZoom`** — expensive; defer work.
- **`scrollViewWillBeginZooming`/`scrollViewDidEndZooming`** — expensive; defer work.
- **`scrollViewDidScrollToTop`** — expensive; defer work.
- **`scrollViewShouldScrollToTop`** — expensive; defer work.

### AppKit performance
- **`NSView`** — efficient layout; use `NSStackView` for stacks.
- **`NSLayoutConstraint`** — efficient layout; avoid `layout`/`layoutSubviews` heavy work.
- **`NSAnimationContext`** — efficient animations; use `NSAnimationContext.runAnimationGroup`.
- **`CALayer`** — efficient rendering; use `shouldRasterize` for complex layers.
- **`drawRect:`** — expensive; use `CAShapeLayer`/`CAGradientLayer` instead.
- **`layout`/`layoutSubviews`** — expensive; minimize work.
- **`updateConstraints`** — efficient constraint updates.
- **`viewWillAppear`/`viewDidAppear`** — expensive; defer work.
- **`viewWillDisappear`/`viewDidDisappear`** — expensive; defer work.
- **`viewDidLoad`** — expensive; defer work.
- **`viewDidLayout`** — expensive; defer work.
- **`prepareForReuse`** — cell reuse; reset state.
- **`cellForItemAt`** — cell creation; reuse cells.
- **`scrollViewDidScroll`** — expensive; throttle updates.
- **`scrollViewWillBeginDragging`/`scrollViewDidEndDragging`** — expensive; defer work.
- **`scrollViewWillBeginDecelerating`/`scrollViewDidEndDecelerating`** — expensive; defer work.
- **`scrollViewDidEndScrollingAnimation`** — expensive; defer work.
- **`scrollViewDidZoom`** — expensive; defer work.
- **`scrollViewWillBeginZooming`/`scrollViewDidEndZooming`** — expensive; defer work.
- **`scrollViewDidScrollToTop`** — expensive; defer work.
- **`scrollViewShouldScrollToTop`** — expensive; defer work.
- **`scrollViewWillBeginDragging`/`scrollViewDidEndDragging`** — expensive; defer work.
- **`scrollViewWillBeginDecelerating`/`scrollViewDidEndDecelerating`** — expensive; defer work.
- **`scrollViewDidEndScrollingAnimation`** — expensive; defer work.
- **`scrollViewDidZoom`** — expensive; defer work.
- **`scrollViewWillBeginZooming`/`scrollViewDidEndZooming`** — expensive; defer work.
- **`scrollViewDidScrollToTop`** — expensive; defer work.
- **`scrollViewShouldScrollToTop`** — expensive; defer work.

## 5. Scrolling performance

- **`UITableView`/`UICollectionView`** — reuse cells, `diffableDataSource`.
- **`NSCollectionView`** — reuse cells, `NSDiffableDataSource`.
- **`ScrollView`** — lazy loading for large content.
- **`LazyVStack`/`LazyHStack`** — lazy loading for stacks.
- **`List`** — lazy loading for lists.
- **`ForEach` + `Identifiable`** — stable identity for diffing.
- **`prefetchDataSource`** — prefetch data for smooth scrolling.
- **`willDisplay`** — prefetch data for cell.
- **`didEndDisplaying`** — cleanup for cell.
- **`prepareForReuse`** — reset cell state.
- **`scrollViewDidScroll`** — throttle updates, avoid heavy work.
- **`CADisplayLink`** — frame-synced updates; use sparingly.
- **`DispatchQueue.main.async`** — defer work to next runloop.
- **`Task.detached`** — defer work to background.
- **`DispatchQueue.global`** — background queue for heavy work.
- **`DispatchQueue.main`** — main queue for UI updates.
- **`DispatchGroup`** — coordinate multiple async operations.
- **`DispatchSemaphore`** — coordinate threads; use sparingly.
- **`NSOperationQueue`** — operation queue for background work.
- **`NSOperation`** — operation for background work.
- **`NSBlockOperation`** — block operation for background work.
- **`NSInvocationOperation`** — invocation operation for background work.
- **`NSOperationQueue`** — queue for operations.
- **`NSOperation`** — operation.
- **`NSBlockOperation`** — block operation.
- **`NSInvocationOperation`** — invocation operation.
- **`NSOperationQueue`** — queue.
- **`NSOperation`** — operation.
- **`NSBlockOperation`** — block operation.
- **`NSInvocationOperation`** — invocation operation.
- **`NSOperationQueue`** — queue.
- **`NSOperation`** — operation.
- **`NSBlockOperation`** — block operation.
- **`NSInvocationOperation`** — invocation operation.
- **`NSOperationQueue`** — queue.
- **`NSOperation`** — operation.
- **`NSBlockOperation`** — block operation.
- **`NSInvocationOperation`** — invocation operation.
- **`NSOperationQueue`** — queue.

## 6. Common performance bugs → fixes

| Bug | Cause | Fix |
|---|---|---|
| Slow launch | heavy `viewDidLoad`, many dylibs | defer work, reduce dylibs, use `DYLD_PRINT_STATISTICS` |
| Memory spike | large arrays, image loading | stream data, use `NSCache`, downsample images |
| Retain cycle | closure captures `self` | use `[weak self]`, `unowned` for non-optional |
| SwiftUI re-render storm | `@State`/`@Binding` overuse | use `@Observable`, `Equatable`, `LazyVStack` |
| Scroll jank | heavy `cellForItemAt`, `scrollViewDidScroll` | reuse cells, throttle updates, prefetch |
| Battery drain | frequent location/network updates | reduce frequency, use `desiredAccuracy`, batch requests |
| Thermal throttle | sustained CPU/GPU usage | reduce work, use `Task.detached`, lower quality |
| Network slow | no caching, large payloads | use `URLSession` caching, compress, paginate |
| Database slow | unindexed queries, large fetches | add indexes, paginate, use `NSFetchRequest` |
| Image loading slow | full-resolution images | downsample, use `UIImage`/`NSImage` caching |
| Font loading slow | custom fonts per view | cache fonts, use `UIFont`/`NSFont` |
| String formatting slow | `String(format:)` in loops | use `String` interpolation, cache formatters |
| JSON parsing slow | large JSON, no streaming | use `JSONDecoder`, stream with `JSONSerialization` |
| Regex slow | complex patterns, backtracking | simplify patterns, use `NSRegularExpression` caching |
| Date formatting slow | `DateFormatter` per call | reuse formatter, cache instances |
| File I/O slow | synchronous reads/writes | use `FileHandle`, `DispatchQueue`, async I/O |
| Core Data slow | unindexed fetches, large contexts | add indexes, use `NSFetchRequest`, batch operations |
| SwiftData slow | unindexed queries, large contexts | add indexes, use `ModelContext`, batch operations |
| Metal slow | too many draw calls, state changes | batch draws, minimize state changes, use instancing |
| RealityKit slow | too many entities, complex physics | reduce entities, simplify physics, use LOD |
| ARKit slow | high-resolution tracking, complex scene | reduce resolution, simplify scene, use `frameSemantics` |
| Vision slow | large images, complex requests | downsample, use `VNImageRequestHandler` caching |
| CoreML slow | large models, complex inference | quantize, use `MLModelConfiguration`, batch inference |
| Speech slow | real-time recognition, complex audio | use `SFSpeechAudioBufferRecognitionRequest`, optimize audio |
| HealthKit slow | frequent queries, large datasets | batch queries, use `HKAnchoredObjectQuery` |
| HomeKit slow | frequent accessory updates, large data | batch updates, use `HMAccessory` caching |
| MapKit slow | frequent region changes, large annotations | batch annotations, use `MKMapSnapshotter` |
| Contacts slow | frequent fetches, large datasets | batch fetches, use `CNContactStore` caching |
| EventKit slow | frequent event queries, large datasets | batch queries, use `EKEventStore` caching |
| Photos slow | frequent fetches, large datasets | batch fetches, use `PHPhotoLibrary` caching |
| AVFoundation slow | frequent asset loads, large files | cache assets, use `AVAssetResourceLoader` |
| CoreMedia slow | frequent sample buffer creation | cache sample buffers, use `CMSampleBufferCreate` |
| CoreVideo slow | frequent pixel buffer creation | cache pixel buffers, use `CVPixelBufferCreate` |
| CoreImage slow | frequent filter applications | cache filters, use `CIContext` caching |
| CoreGraphics slow | frequent context creation | cache contexts, use `CGContext` caching |
| CoreAnimation slow | frequent layer updates | batch updates, use `CALayer` caching |
| QuartzCore slow | frequent render server calls | batch updates, use `CALayer` caching |
| UIKit slow | frequent view updates, heavy layout | batch updates, use `UIViewPropertyAnimator` |
| AppKit slow | frequent view updates, heavy layout | batch updates, use `NSAnimationContext` |
| SwiftUI slow | frequent body recomputation | use `@Observable`, `Equatable`, `LazyVStack` |
| Combine slow | frequent publisher emissions | debounce, throttle, use `share`/`multicast` |
| Foundation slow | frequent string/date formatting | cache formatters, use `String` interpolation |
| Swift slow | frequent allocations, boxing | use value types, avoid `AnyObject` boxing |
| Dispatch slow | frequent queue hops | batch work, use `DispatchGroup` |
| os_log slow | frequent logging | reduce log volume, use `.private` redaction |
| os_signpost slow | frequent signposts | reduce signpost volume, use `OSSignposter` caching |
| os_activity slow | frequent activities | reduce activity volume, use `OSActivity` caching |
| os_trace slow | frequent traces | reduce trace volume, use `OSTrace` caching |
| os_telemetry slow | frequent telemetry | reduce telemetry volume, use `OSTelemetry` caching |
| os_metrics slow | frequent metrics | reduce metrics volume, use `OSMetrics` caching |
