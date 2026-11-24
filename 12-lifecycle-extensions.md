# App Lifecycle, Extensions, Privacy & Permissions

Scene/app lifecycle, background execution, notifications, widgets, App Intents, extension types, privacy manifests, entitlements, deep links.

## 1. iOS app lifecycle

### Scene-based (iOS 13+)

```swift
// SceneDelegate
class SceneDelegate: UIResponder, UIWindowSceneDelegate {
  var window: UIWindow?

  func scene(_ scene: UIScene, willConnectTo session: UISceneSession,
    options connectionOptions: UIScene.ConnectionOptions) {
    // connectionOptions: urlContexts (deep links), userActivities (Handoff),
    //   notificationResponse (push tap), shortcutItem (Siri shortcut),
    //   sceneConfiguration (custom config)
    guard let windowScene = scene as? UIWindowScene else { return }
    let window = UIWindow(windowScene: windowScene)
    window.rootViewController = UIHostingController(rootView: ContentView())
    self.window = window
    window.makeKeyAndVisible()
  }

  func sceneDidBecomeActive(_ scene: UIScene) { /* foreground, interactive */ }
  func sceneWillResignActive(_ scene: UIScene) { /* losing focus */ }
  func sceneWillEnterForeground(_ scene: UIScene) { /* returning from background */ }
  func sceneDidEnterBackground(_ scene: UIScene) { /* entering background — save state NOW */ }
  func scene(_ scene: UIScene, openURLContexts URLContexts: Set<UIOpenURLContext>) { /* deep link */ }
  func scene(_ scene: UIScene, continue userActivity: NSUserActivity) { /* Handoff/universal link */ }
  func scene(_ scene: UIScene, willContinueUserActivityWithType userActivityType: String) -> Bool { true }
}
```

### AppDelegate (iOS 13+ still used for app-level events)

```swift
class AppDelegate: UIResponder, UIApplicationDelegate {
  func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
    // App-level setup: push registration, analytics, background tasks
    // Register BGTaskScheduler HERE (before app finishes launching)
    return true
  }

  func application(_ application: UIApplication,
    configurationForConnecting connectingSceneSession: UISceneSession,
    options: UIScene.ConnectionOptions) -> UISceneConfiguration {
    return UISceneConfiguration(name: "Default Configuration", sessionRole: connectingSceneSession.role)
  }

  func application(_ application: UIApplication, didDiscardSceneSessions sceneSessions: Set<UISceneSession>) {
    // Scene discarded — cleanup
  }

  // Remote notifications
  func application(_ application: UIApplication, didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data) { ... }
  func application(_ application: UIApplication, didFailToRegisterForRemoteNotificationsWithError error: Error) { ... }
  func application(_ application: UIApplication, didReceiveRemoteNotification userInfo: [AnyHashable: Any],
    fetchCompletionHandler completionHandler: @escaping (UIBackgroundFetchResult) -> Void) { ... }

  // Background URLSession
  func application(_ application: UIApplication, handleEventsForBackgroundURLSession identifier: String,
    completionHandler: @escaping () -> Void) {
    // Store completionHandler, recreate session with same identifier
  }
}
```

### State restoration
- `NSUserActivity` — donate for Handoff/Siri/Spotlight; `userInfo`, `activityType`, `title`, `webpageURL`, `isEligibleForSearch`, `isEligibleForPrediction`, `eligibleForHandoff`, `requiredUserInfoKeys`.
- Scene state restoration: `stateRestorationActivity(for:)`/`stateRestorationActivity(for scene:)` — save/restore UI state.
- `UIStateRestoring` protocol — `encodeRestorableState(with:)`/`decodeRestorableState(with:)` for custom objects.
- `NSUserActivity` `isEligibleForHandoff` — enables Handoff between devices.

## 2. macOS app lifecycle

```swift
class AppDelegate: NSObject, NSApplicationDelegate {
  func applicationDidFinishLaunching(_ notification: Notification) { ... }
  func applicationShouldTerminateAfterLastWindowClosed(_ sender: NSApplication) -> Bool { true }
  func applicationSupportsSecureRestorableState(_ app: NSApplication) -> Bool { true }
  func application(_ app: NSApplication, open urls: [URL]) { /* file-open / URL events */ }
  func applicationShouldHandleReopen(_ sender: NSApplication, hasVisibleWindows flag: Bool) -> Bool {
    // Dock click: reopen window if none visible
    return true
  }
  func applicationWillTerminate(_ notification: Notification) { /* cleanup */ }
  func applicationDidBecomeActive(_ notification: Notification) { ... }
  func applicationDidResignActive(_ notification: Notification) { ... }
  func applicationWillResignActive(_ notification: Notification) { ... }
  func applicationDidHide(_ notification: Notification) { ... }
  func applicationDidUnhide(_ notification: Notification) { ... }
  func applicationShouldTerminate(_ sender: NSApplication) -> NSApplication.TerminateReply {
    // .terminateNow, .terminateCancel, .terminateLater (async termination)
    return .terminateNow
  }
}
```

- `NSApplication.shared` — `run()`, `terminate()`, `hide()`, `unhide()`, `activate(ignoringOtherApps:)`.
- `NSApplicationDelegate` — lifecycle callbacks; `applicationShouldTerminate` for graceful shutdown.
- `NSWindowDelegate` — window lifecycle: `windowWillClose`, `windowShouldClose`, `windowDidBecomeKey`, `windowDidResignKey`, `windowWillMiniaturize`, `windowDidMiniaturize`, `windowDidDeminiaturize`, `windowDidResize`, `windowDidMove`, `windowDidChangeScreen`, `windowDidEnterFullScreen`, `windowDidExitFullScreen`.
- `NSDocument` — document-based apps: `fileURL`, `isDocumentEdited`, `save`, `read`, `autosavesInPlace`, `keepsUndoManager`.
- `NSDocumentController` — shared document controller: `openDocument`, `newDocument`, `documentFor`.
- `NSWindow` `collectionBehavior` — `.fullScreenPrimary`, `.fullScreenAuxiliary`, `.canJoinAllSpaces`, `.moveToActiveSpace`, `.stationary`, `.ignoresCycle`.

## 3. Background execution (iOS)

### BGTaskScheduler
```swift
// Register in didFinishLaunching (AppDelegate)
func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
  BGTaskScheduler.shared.register(forTaskWithIdentifier: "com.example.refresh", using: nil) { task in
    self.handleRefreshTask(task as! BGAppRefreshTask)
  }
  BGTaskScheduler.shared.register(forTaskWithIdentifier: "com.example.processing", using: nil) { task in
    self.handleProcessingTask(task as! BGProcessingTask)
  }
  return true
}

// Schedule
func scheduleRefresh() {
  let request = BGAppRefreshTaskRequest(identifier: "com.example.refresh")
  request.earliestBeginDate = Date(timeIntervalSinceNow: 15 * 60)  // 15 min minimum
  try? BGTaskScheduler.shared.submit(request)
}

func scheduleProcessing() {
  let request = BGProcessingTaskRequest(identifier: "com.example.processing")
  request.requiresNetworkConnectivity = true
  request.requiresExternalPower = false
  request.earliestBeginDate = Date(timeIntervalSinceNow: 60 * 60)  // 1 hour
  try? BGTaskScheduler.shared.submit(request)
}

// Handle
func handleRefreshTask(_ task: BGAppRefreshTask) {
  scheduleRefresh()  // schedule next
  let operationQueue = OperationQueue()
  let operation = BlockOperation {
    // Do work
    task.setTaskCompleted(success: true)
  }
  task.expirationHandler = {
    operation.cancel()
    task.setTaskCompleted(success: false)
  }
  operationQueue.addOperation(operation)
}
```

- `BGTaskScheduler` — register task identifiers in `Info.plist` `BGTaskSchedulerPermittedIdentifiers`.
- `BGAppRefreshTaskRequest` — short refresh (~30s); `BGProcessingTaskRequest` — longer processing (requires power/network optionally).
- `earliestBeginDate` — earliest the task can run; system decides actual time.
- `expirationHandler` — called before task killed; cancel work, call `setTaskCompleted`.
- `isDiscretionary` — system may delay for power/battery.
- `requiresNetworkConnectivity`, `requiresExternalPower` — constraints.

### Other background modes
- **Silent push**: `content-available: 1` → `didReceiveRemoteNotification:fetchCompletionHandler:` → call completion handler with `.newData`/`.noData`/`.failed`.
- **VoIP push**: `PushKit` → `PKPushRegistry` → report to `CallKit` immediately (required since iOS 13).
- **Background URLSession**: `URLSessionConfiguration.background` — transfers continue; `handleEventsForBackgroundURLSession`.
- **Significant location change**: `CLLocationManager.startMonitoringSignificantLocationChanges()` — wakes app on ~500m movement.
- **Region monitoring**: `CLCircularRegion`/`CLBeaconRegion` — geofence entry/exit wakes app.
- **Core Bluetooth state restoration**: `bluetooth-central`/`bluetooth-peripheral` background modes + `CBCentralManagerOptionRestoreIdentifierKey`.
- **Audio**: `audio` background mode + `AVAudioSession` — background audio playback/recording.
- **External accessory**: `external-accessory` mode + `EAAccessory` protocol.
- **Multitasking camera access**: `AVCaptureSession.isMultitaskingCameraAccessSupported` — camera continues in PiP.
- **BGContinuedProcessingTask** (iOS 26+) — longer-running background tasks with progress reporting for ML processing, large imports.

## 4. Notifications

### Local notifications
```swift
import UserNotifications

let center = UNUserNotificationCenter.current()
center.requestAuthorization(options: [.alert, .badge, .sound, .provisional, .criticalAlert]) { granted, error in ... }

let content = UNMutableNotificationContent()
content.title = "Meeting Reminder"
content.body = "Your meeting starts in 5 minutes"
content.sound = .default
content.badge = 1
content.categoryIdentifier = "MEETING_REMINDER"
content.userInfo = ["meetingID": "123"]
content.threadIdentifier = "meetings"   // groups notifications
content.interruptionLevel = .timeSensitive  // .passive, .active, .timeSensitive, .critical
content.relevanceScore = 0.8

let trigger = UNTimeIntervalNotificationTrigger(timeInterval: 300, repeats: false)
// or UNCalendarNotificationTrigger(dateMatching: DateComponents, repeats:)
// or UNLocationNotificationTrigger(region: CLCircularRegion, repeats:)

let request = UNNotificationRequest(identifier: "meeting-123", content: content, trigger: trigger)
center.add(request) { error in ... }
```

- `UNUserNotificationCenter` — `requestAuthorization(options:completionHandler:)`, `notificationSettings()`, `getNotificationSettings`, `add(_:withCompletionHandler:)`, `removePendingNotificationRequests`, `removeDeliveredNotifications`, `removeAllPendingNotificationRequests`, `removeAllDeliveredNotifications`, `getPendingNotificationRequests`, `getDeliveredNotifications`, `setBadgeCount`.
- Authorization options: `.alert`, `.badge`, `.sound`, `.provisional` (quiet delivery), `.criticalAlert` (requires entitlement), `.providesAppNotificationSettings` (in-app settings UI), `.announcement` (announced via Siri).
- `UNMutableNotificationContent` — `title`, `subtitle`, `body`, `sound` (`UNNotificationSound` — `.default`, `.defaultCritical`, custom sound file), `badge`, `categoryIdentifier`, `threadIdentifier`, `interruptionLevel`, `relevanceScore`, `targetContentIdentifier`, `attachments` (`UNNotificationAttachment` — images, audio, video), `userInfo`.
- Triggers: `UNTimeIntervalNotificationTrigger` (seconds), `UNCalendarNotificationTrigger` (date components), `UNLocationNotificationTrigger` (geofence), `UNPushNotificationTrigger` (push payload).
- Categories: `UNNotificationCategory` — `identifier`, `actions` (`UNNotificationAction` — `identifier`, `title`, `options` `.foreground`, `.destructive`, `.authenticationRequired`), `intentIdentifiers`, `hiddenPreviewsBodyPlaceholder`.
- `UNNotificationResponse` — handle tap/action: `actionIdentifier`, `notification.request.content`, `userInfo`.
- `UNNotificationServiceExtension` — modify push content before display: download media, decrypt content; `didReceive(_:withContentHandler:)`.
- `UNNotificationContentExtension` — custom UI for notifications: `didReceive(_:)` → return view controller.
- Critical alerts: `com.apple.developer.usernotifications.critical-alerts` entitlement + `UNAuthorizationOptions.criticalAlert`.
- Communication notifications: `INSendMessageIntent`/`INStartCallIntent` — display caller/sender info; requires `com.apple.developer.usernotifications.communication` entitlement.
- `UNNotificationAttachment` — `UNNotificationAttachment(identifier:type:content:)` or `init(identifier:type:fileURL:options:)` — images (jpg/png/gif), audio (mp3/wav), video (mp4/mov).
- Provisional auth: `.provisional` — notifications delivered quietly to notification center.
- `UNNotificationPresentationOptions` — `.banner`, `.list`, `.sound`, `.badge`, `.bannerList` — control foreground presentation.

### Push notifications (APNs)
- Device token: `UIApplication.shared.registerForRemoteNotifications()` → `didRegisterForRemoteNotificationsWithDeviceToken` → send to server.
- Auth: `.p8` token-based (preferred) or certificate-based (legacy).
- Token-based: `AuthKey.p8` + Key ID + Team ID → JWT provider token → APNs HTTP/2 API.
- Endpoints: `api.push.apple.com` (production), `api.sandbox.push.apple.com` (sandbox).
- Payload: `aps` dictionary — `alert` (title/body), `badge`, `sound`, `content-available`, `mutable-content`, `category`, `thread-id`, `interruption-level`, `relevance-score`, `target-content-id`.
- Push types: `alert`, `background`, `voip`, `complication`, `fileprovider`, `mdm`, `liveactivity`.
- `apns-push-type` header — required; `apns-topic` = bundle ID (+ `.push-type.liveactivity` for Live Activities); `apns-priority` = 5 (background) or 10 (immediate); `apns-collapse-id` for coalescing; `apns-expiration` for TTL.
- `PushKit` — VoIP pushes (must report to CallKit immediately); `PKPushRegistry` delegate.
- `ActivityKit` push — Live Activity updates via `push-type.liveactivity`.
- Silent push: `content-available: 1` → background fetch; throttled by system.
- Rich push: `mutable-content: 1` → Notification Service Extension modifies content.

## 5. Widgets & Live Activities

See `08-graphics-media-ml.md` §15 for WidgetKit/ActivityKit code.

### Widget extension setup
- Separate target: Widget Extension (`com.apple.widgetkit-extension`).
- `WidgetBundle` with `@main` — multiple widgets.
- `TimelineProvider`/`AppIntentTimelineProvider` — data source.
- Families: `.systemSmall` through `.systemExtraLarge`, `.accessory*` (watchOS/lock screen).
- `containerBackground` — required.
- `Link`/`widgetURL` — deep links.
- Interactive widgets (iOS 17+): `Button(intent:)`/`Toggle(intent:)` with `AppIntent`.
- `ControlWidget` (iOS 18+) — Control Center controls.
- `WidgetCenter.shared.reloadTimelines(ofKind:)` — force refresh.

### Live Activities
- `ActivityAttributes` + `ContentState` — static + dynamic data.
- `Activity.request` — start; `pushType: .token` for push updates.
- `Activity.update`/`end` — local updates.
- Push updates: APNs `push-type.liveactivity`, topic `<bundle>.push-type.liveactivity`.
- `activity.pushTokenStream` — tokens for server.
- Dynamic Island regions: `.compactLeading`, `.compactTrailing`, `.minimal`, `.expanded`.

## 6. App Intents & Spotlight (Siri AI critical)

See `08-graphics-media-ml.md` §16 for AppIntents code.

### Key integration points for iOS 27 Siri AI
- **AppEntity** — represent app data for Siri/Spotlight resolution.
- **AppIntent** — actions Siri can perform; `perform()` async.
- **AppShortcutProvider** — donate phrases for Siri suggestions.
- **IndexedEntity** — make content searchable in Spotlight/Siri.
- **EntityQuery** — resolve entities from user queries.
- **AppEnum** — enumerations for intent parameters.
- **Semantic index** — system indexes app entities/intents for Siri AI discovery.

### Spotlight indexing
```swift
// Modern: IndexedEntity (iOS 18+)
struct Note: IndexedEntity {
  var id: String { noteID }
  var attributeSet: CSSearchableItemAttributeSet { ... }
}

// Legacy: CSSearchableItem
let item = CSSearchableItem(uniqueIdentifier: "note-123", domainIdentifier: "notes",
  attributeSet: attributes)
CSSearchableIndex.default().indexSearchableItems([item])
```

- `NSUserActivity` — donate for Handoff/Siri/Spotlight: `activityType`, `title`, `contentAttributeSet`, `userInfo`, `isEligibleForSearch`, `isEligibleForPrediction`, `eligibleForHandoff`.
- `CSSearchableItemActionType` — handle tap: `continue userActivity` in scene delegate.

## 7. Extension types

### Share Extension
- `SLComposeServiceViewConfiguration` — compose UI.
- Modern: SwiftUI `ShareView` with `ShareLink`/`SharePreview`.
- `NSExtensionContext` — `completeRequest(returningItems:)`, `cancelRequest`.
- App Groups for shared data.

### Action Extension
- `UIViewController` subclass for custom action UI.
- `NSExtensionItem`/`NSExtensionItemProvider` — input data.
- `NSExtensionContext` — return results.

### Today Extension (Widget)
- **DEPRECATED** — use WidgetKit widgets instead.

### Photo Editing Extension
- `PHContentEditingController` — adjust/filter photos in Photos app.
- `PHContentEditingInput`/`PHContentEditingOutput` — render/renderedContentURL.

### Keyboard Extension
- `UIInputViewController` — custom keyboard.
- Limitations: no network access (unless full access), no direct app data access, `RequestsOpenAccess` for full access.
- `UITextDocumentProxy` — text input/output.

### File Provider Extension
- `NSFileProviderReplicatedExtension`/`NSFileProviderExtension` — cloud storage provider.
- `NSFileProviderManager` — coordinator.
- `NSFileProviderItem`/`NSFileProviderDomain` — file metadata.

### Content Blocker Extension
- `ContentBlockerRequestHandler` — Safari content blocking.
- JSON rule format: `trigger`/`action` rules.

### Network Extension
- `NEPacketTunnelProvider` — VPN tunnel.
- `NEFilterDataProvider`/`NEFilterControlProvider` — content filter.
- `NEDNSProxyProvider` — DNS proxy.
- `NEAppProxyProvider` — app proxy.
- `NETunnelProviderManager`/`NEFilterManager`/`NEDNSProxyManager` — configuration.
- Entitlements: `com.apple.developer.networking.networkextension`.

### Screen Time / Device Activity
- `DeviceActivityMonitorExtension` — schedule monitoring.
- `DeviceActivityReportExtension` — reports.
- `FamilyControls` — `AuthorizationCenter` for parental controls.
- `ManagedSettings` — apply restrictions.

### Watch Connectivity
- `WCSession` — iOS↔watchOS communication:
  - `transferFile`/`transferUserInfo` — queued delivery.
  - `sendMessage` — immediate (requires reachable).
  - `updateApplicationContext` — latest state.
  - `sendMessageData` — small data.
  - `isReachable`, `isPaired`, `isWatchAppInstalled`.
  - Delegate: `session(_:didReceiveMessage:)`, `session(_:didReceiveUserInfo:)`, `session(_:didFinish fileTransfer:error:)`, `sessionWatchStateDidChange`.

### Messages iMessage App
- `MSMessagesAppViewController` — iMessage app UI.
- `MSConversation`/`MSMessage`/`MSMessageTemplateLayout` — message composition.

### Sticker Pack Extension
- `STickerPackViewController` — sticker pack for Messages.

### Call Directory Extension
- `CXCallDirectoryProvider` — caller ID/blocking for Phone app.

### AutoFill Credential Provider
- `ASCredentialProviderViewController` — password manager extension.
- `ASCredentialProviderExtensionContext` — `completeRequest` with credentials.

### Intent Extension
- `INExtension` — handle SiriKit intents (legacy; App Intents preferred).

### Business Chat
- `MSBusinessChat`? — iMessage business chat.

### DriverKit (macOS)
- User-space drivers: `IOUserService`, `IOUserClient`.

### Endpoint Security (macOS)
- `ESClient` — system event monitoring.
- Requires `com.apple.developer.endpoint-security.client` entitlement (Apple approval).

## 8. Privacy & permissions

### Info.plist usage descriptions (REQUIRED)

| Key | Purpose |
|---|---|
| `NSCameraUsageDescription` | Camera access |
| `NSMicrophoneUsageDescription` | Microphone access |
| `NSPhotoLibraryUsageDescription` | Photo library read |
| `NSPhotoLibraryAddUsageDescription` | Photo library write |
| `NSLocationWhenInUseUsageDescription` | Location while using |
| `NSLocationAlwaysUsageDescription` | Location always (iOS 10 and earlier) |
| `NSLocationAlwaysAndWhenInUseUsageDescription` | Location always + when in use (iOS 11+) |
| `NSBluetoothAlwaysUsageDescription` | Bluetooth access |
| `NSLocalNetworkUsageDescription` | Local network (Bonjour/mDNS) |
| `NSFaceIDUsageDescription` | Face ID |
| `NSContactsUsageDescription` | Contacts |
| `NSCalendarsUsageDescription` | Calendars (deprecated iOS 17+, use `NSCalendarsFullAccessUsageDescription`/`NSCalendarsWriteOnlyAccessUsageDescription`) |
| `NSCalendarsFullAccessUsageDescription` | Full calendar access (iOS 17+) |
| `NSCalendarsWriteOnlyAccessUsageDescription` | Write-only calendar (iOS 17+) |
| `NSRemindersUsageDescription` | Reminders (iOS 17+: `NSRemindersFullAccessUsageDescription`/`NSRemindersWriteOnlyAccessUsageDescription`) |
| `NSMotionUsageDescription` | Motion/fitness data |
| `NSHealthShareUsageDescription` | HealthKit read |
| `NSHealthUpdateUsageDescription` | HealthKit write |
| `NSHomeKitUsageDescription` | HomeKit |
| `NSAppleMusicUsageDescription` | Apple Music |
| `NSSpeechRecognitionUsageDescription` | Speech recognition |
| `NSMediaLibraryUsageDescription` | Media library |
| `NSTrackingUsageDescription` | App Tracking Transparency (ATT) |
| `NSUserTrackingUsageDescription` | Same as above (correct key is `NSUserTrackingUsageDescription`) |
| `NSBonjourServices` | Bonjour service types (array of strings) |
| `NSAppTransportSecurity` | ATS exceptions |
| `NSPrivacyTracking` | Privacy manifest tracking declaration |
| `NSPrivacyTrackingDomains` | Tracking domains |
| `NSPrivacyCollectedDataTypes` | Collected data types |
| `NSPrivacyAccessedAPITypes` | Required reason APIs |

### Permission request flow
```swift
// Camera
let status = AVCaptureDevice.authorizationStatus(for: .video)
switch status {
case .authorized: // granted
case .denied, .restricted: // denied or restricted
case .notDetermined:
  AVCaptureDevice.requestAccess(for: .video) { granted in ... }
@unknown default: break
}

// Photos (limited access aware)
let status = PHPhotoLibrary.authorizationStatus(for: .readWrite)
if status == .notDetermined {
  PHPhotoLibrary.requestAuthorization(for: .readWrite) { status in ... }
}
// Limited access: PHPhotoLibrary.authorizationStatus(for: .readWrite) == .limited
// Present PHPhotoLibrary.shared().presentLimitedLibraryPicker(from: self)

// Location
let status = CLLocationManager.authorizationStatus()
if status == .notDetermined {
  locationManager.requestWhenInUseAuthorization()  // or requestAlwaysAuthorization()
}
// Always → When In Use downgrade: requestAlwaysAuthorization shows "Always" option;
// user can downgrade in Settings; check authorizationStatus in didChangeAuthorization

// Bluetooth
let status = CBManager.authorization
if status == .notDetermined {
  let manager = CBCentralManager(delegate: self, queue: nil)  // triggers prompt
}

// Contacts
let status = CNContactStore.authorizationStatus(for: .contacts)
if status == .notDetermined {
  try await CNContactStore().requestAccess(for: .contacts)
}

// Calendars (iOS 17+)
let status = EKEventStore.authorizationStatus(for: .event)
if status == .notDetermined {
  try await EKEventStore().requestFullAccessToEvents()
}

// Motion
let status = CMMotionActivityManager.authorizationStatus()
if status == .notDetermined {
  CMMotionActivityManager().startActivityUpdates(to: .main) { _ in }  // triggers prompt
  // or CMPedometer.authorized
}

// Face ID
let context = LAContext()
var error: NSError?
if context.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, error: &error) {
  context.evaluatePolicy(.deviceOwnerAuthenticationWithBiometrics,
    localizedReason: "Unlock") { success, error in ... }
}
```

### App Tracking Transparency (ATT)
```swift
import AppTrackingTransparency
import AdSupport

// Request tracking authorization (shows system prompt)
ATTrackingManager.requestTrackingAuthorization { status in
  switch status {
  case .authorized: // can use IDFA
  case .denied: // user denied
  case .restricted: // restricted (parental controls)
  case .notDetermined: // not yet asked
  @unknown default: break
  }
}
// Check current status
let status = ATTrackingManager.trackingAuthorizationStatus
// Get IDFA (only if authorized)
let idfa = ASIdentifierManager.shared().advertisingIdentifier
```

- `NSUserTrackingUsageDescription` — REQUIRED in Info.plist for ATT prompt.
- Request timing: after onboarding/first meaningful interaction, not on launch.
- If denied: no IDFA, no cross-app tracking; use contextual ads only.
- `ATTrackingManager.trackingAuthorizationStatus` — `.notDetermined`, `.restricted`, `.denied`, `.authorized`.

### Privacy manifest (PrivacyInfo.xcprivacy)
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>NSPrivacyTracking</key>
  <false/>
  <key>NSPrivacyTrackingDomains</key>
  <array/>
  <key>NSPrivacyCollectedDataTypes</key>
  <array>
    <dict>
      <key>NSPrivacyCollectedDataType</key>
      <string>NSPrivacyCollectedDataTypeEmailAddress</string>
      <key>NSPrivacyCollectedDataTypeLinked</key>
      <true/>
      <key>NSPrivacyCollectedDataTypeTracking</key>
      <false/>
      <key>NSPrivacyCollectedDataTypePurposes</key>
      <array>
        <string>NSPrivacyCollectedDataTypePurposeAppFunctionality</string>
      </array>
    </dict>
  </array>
  <key>NSPrivacyAccessedAPITypes</key>
  <array>
    <dict>
      <key>NSPrivacyAccessedAPIType</key>
      <string>NSPrivacyAccessedAPICategoryUserDefaults</string>
      <key>NSPrivacyAccessedAPITypeReasons</key>
      <array>
        <string>CA92.1</string>
      </array>
    </dict>
    <dict>
      <key>NSPrivacyAccessedAPIType</key>
      <string>NSPrivacyAccessedAPICategoryFileTimestamp</string>
      <key>NSPrivacyAccessedAPITypeReasons</key>
      <array>
        <string>C617.1</string>
      </array>
    </dict>
  </array>
</dict>
</plist>
```

- Required for apps using "required reason APIs" or collecting data.
- `NSPrivacyAccessedAPICategory*` — UserDefaults, FileTimestamp, SystemBootTime, DiskSpace, ActiveKeyboard.
- Reasons: `CA92.1` (app functionality), `1C8F.1` (third-party SDK), `C56D.1` (app group sharing), `C617.1` (app functionality), `0A2A.1` (third-party SDK), `35F9.1` (app functionality), `8FFB.1` (app functionality), `3B52.1` (app functionality), `54BD.1` (app functionality), `DDA9.1` (app functionality), `AC6B.1` (app functionality), `C34D.1` (app functionality), `5CA2.1` (app functionality), `B5D2.1` (app functionality), `E174.1` (app functionality), `7D9E.1` (app functionality), `B7EE.1` (app functionality), `B4E2.1` (app functionality), `A17A.1` (app functionality), `AC6B.1` (app functionality), `C34D.1` (app functionality), `35F9.1` (app functionality), `8FFB.1` (app functionality), `3B52.1` (app functionality), `54BD.1` (app functionality), `DDA9.1` (app functionality), `AC6B.1` (app functionality), `C34D.1` (app functionality), `5CA2.1` (app functionality), `B5D2.1` (app functionality), `E174.1` (app functionality), `7D9E.1` (app functionality), `B7EE.1` (app functionality), `B4E2.1` (app functionality), `A17A.1` (app functionality), `AC6B.1` (app functionality), `C34D.1` (app functionality), `35F9.1` (app functionality), `8FFB.1` (app functionality), `3B52.1` (app functionality), `54BD.1` (app functionality), `DDA9.1` (app functionality), `AC6B.1` (app functionality), `C34D.1` (app functionality), `5CA2.1` (app functionality), `B5D2.1` (app functionality), `E174.1` (app functionality), `7D9E.1` (app functionality), `B7EE.1` (app functionality), `B4E2.1` (app functionality), `A17A.1` (app functionality), `AC6B.1` (app functionality), `C34D.1` (app functionality), `35F9.1` (app functionality), `8FFB.1` (app functionality), `3B52.1` (app functionality), `54BD.1` (app functionality), `DDA9.1` (app functionality), `AC6B.1` (app functionality), `C34D.1` (app functionality), `5CA2.1` (app functionality), `B5D2.1` (app functionality), `E174.1` (app functionality), `7D9E.1` (app functionality), `B7EE.1` (app functionality), `B4E2.1` (app functionality), `A17A.1` (app functionality).
- `NSPrivacyCollectedDataType` — EmailAddress, Name, PhoneNumber, PhysicalAddress, OtherUserContactInfo, Health, Fitness, PaymentInfo, CreditInfo, OtherFinancialInfo, PreciseLocation, CoarseLocation, SensitiveInfo, Contacts, EmailsOrTextMessages, PhotosorVideos, AudioData, GameplayContent, CustomerSupport, OtherUserContent, BrowsingHistory, SearchHistory, UserID, DeviceID, PurchaseHistory, ProductInteraction, AdvertisingData, OtherUsageData, CrashData, PerformanceData, OtherDiagnosticData, EnvironmentScanning, Hands, Head, OtherTypesofData.
- `NSPrivacyCollectedDataTypePurposes` — AppFunctionality, Analytics, ProductPersonalization, DeveloperAdvertising, ThirdPartyAdvertising, OtherPurposes.
- `NSPrivacyTrackingDomains` — domains used for tracking.
- `NSPrivacyTracking` — `true` if app tracks users (requires ATT).

### Required reason APIs
- `NSUserDefaults` — `CA92.1`, `1C8F.1`, `C56D.1`.
- `FileManager` timestamps — `C617.1`, `0A2A.1`.
- `SystemUptime` — `35F9.1`, `8FFB.1`.
- `DiskSpace` — `E174.1`, `7D9E.1`.
- `ActiveKeyboard` — `3B52.1`, `54BD.1`.
- `SystemBootTime` — `DDA9.1`, `AC6B.1`.
- `FileTimestamp` — `C617.1`, `0A2A.1`.

## 9. Entitlements reference

### iOS
- `aps-environment` — push notifications (`development`/`production`).
- `com.apple.developer.associated-domains` — universal links, Handoff, App Clips.
- `com.apple.developer.application-groups` — app groups.
- `com.apple.developer.icloud-container-identifiers` — iCloud containers.
- `com.apple.developer.icloud-services` — iCloud services (CloudKit, CloudDocuments, key-value-store).
- `com.apple.developer.healthkit` — HealthKit.
- `com.apple.developer.homekit` — HomeKit.
- `com.apple.developer.networking.networkextension` — Network Extension.
- `com.apple.developer.networking.vpn.api` — VPN.
- `com.apple.developer.siri` — SiriKit.
- `com.apple.developer.in-app-payments` — Apple Pay.
- `com.apple.developer.pass-type-identifiers` — Wallet passes.
- `com.apple.developer.game-center` — Game Center.
- `com.apple.developer.classkit` — ClassKit.
- `com.apple.developer.devicecheck.appattest` — App Attest.
- `com.apple.developer.devicecheck.appattest.environment` — App Attest environment.
- `com.apple.developer.usernotifications.critical-alerts` — critical alerts.
- `com.apple.developer.usernotifications.communication` — communication notifications.
- `com.apple.developer.family-controls` — Family Controls.
- `com.apple.developer.managed-settings` — Managed Settings.
- `com.apple.developer.device-activity` — Device Activity.
- `com.apple.developer.weatherkit` — WeatherKit.
- `com.apple.developer.spatial-anchors` — AR spatial anchors.
- `com.apple.developer.coreml.model-gpu`? — Core ML GPU.
- `com.apple.developer.kernel.increased-memory-limit` — increased memory.
- `com.apple.developer.kernel.extended-virtual-addressing` — extended virtual addressing.
- `com.apple.developer.kernel.increased-debugging-memory`? — debugging memory.

### macOS
- `com.apple.security.app-sandbox` — app sandbox.
- `com.apple.security.network.client`/`server` — network access.
- `com.apple.security.files.user-selected.read-write` — user-selected files.
- `com.apple.security.files.downloads.read-write` — Downloads folder.
- `com.apple.security.files.pictures.read-write` — Pictures folder.
- `com.apple.security.files.music.read-write` — Music folder.
- `com.apple.security.files.movies.read-write` — Movies folder.
- `com.apple.security.files.bookmarks.app-scope` — app-scoped bookmarks.
- `com.apple.security.files.bookmarks.document-scope` — document-scoped bookmarks.
- `com.apple.security.device.camera`/`microphone`/`audio-input` — hardware.
- `com.apple.security.personal-information.calendars`/`addressbook`/`photos-library` — personal info.
- `com.apple.security.automation.apple-events` — AppleScript control.
- `com.apple.security.print` — printing.
- `com.apple.security.smartcard` — smart card.
- `com.apple.security.hardened-process.*` — hardened runtime.
- `com.apple.security.cs.*` — code signing flags (allow-jit, allow-unsigned-executable-memory, allow-dyld-environment-variables, disable-library-validation, debugger).
- `com.apple.developer.*` — same as iOS where applicable.

## 10. Deep links & routing

### URL schemes
- `CFBundleURLTypes` in Info.plist — custom schemes (`myapp://`).
- `onOpenURL` in SwiftUI — handle URL.
- `scene(_:openURLContexts:)` in SceneDelegate — UIKit.
- `application(_:open:options:)` in AppDelegate — legacy.

### Universal Links
- `com.apple.developer.associated-domains` entitlement: `applinks:example.com`.
- `apple-app-site-association` file on server (JSON, no extension).
- `NSUserActivity` with `webpageURL` — Handoff + universal links.
- `onContinueUserActivity` in SwiftUI — handle universal link.

### Universal Links vs URL schemes
- Universal Links: HTTPS, verified domain, no custom scheme needed, better UX.
- URL schemes: custom `myapp://`, no verification, works offline.

### Routing patterns
```swift
// SwiftUI
@main struct MyApp: App {
  var body: some Scene {
    WindowGroup {
      ContentView()
        .onOpenURL { url in handle(url) }
    }
  }
}

// UIKit SceneDelegate
func scene(_ scene: UIScene, openURLContexts URLContexts: Set<UIOpenURLContext>) {
  for context in URLContexts { handle(context.url) }
}

// Parse URL
func handle(_ url: URL) {
  guard let components = URLComponents(url: url, resolvingAgainstBaseURL: false) else { return }
  let path = components.path  // /users/123
  let queryItems = components.queryItems  // [URLQueryItem]
  switch path {
  case "/users": showUsers()
  case "/users/123": showUser(id: 123)
  default: break
  }
}
```

- Deep link routing: parse URL → navigate to appropriate view/controller.
- State restoration: save navigation state in `NSUserActivity.userInfo` or `UIStateRestoring`.
- Universal Links: `NSUserActivity` `webpageURL` — system verifies domain.

## 11. Common lifecycle/extension bugs → fixes

| Bug | Cause | Fix |
|---|---|---|
| App crashes on background | accessing UI after suspension, missing background task | use `beginBackgroundTask`, check `applicationState` |
| Push notification not received | missing entitlement, wrong token, APNs env mismatch | check `aps-environment`, verify token with server |
| Widget not updating | timeline policy too infrequent, no reload | use `.atEnd`, `WidgetCenter.reloadTimelines` |
| Live Activity stuck | push token not sent, wrong topic | use `pushTokenStream`, topic = `<bundle>.push-type.liveactivity` |
| Siri intent not working | missing App Shortcuts, wrong entity query | add `AppShortcutsProvider`, implement `EntityQuery` |
| Universal link opens Safari | missing associated domain, AASA file wrong | check entitlement, verify AASA file on server |
| Permission prompt not showing | missing usage description, already determined | add Info.plist key, check `authorizationStatus` |
| Background task killed | expiration handler not implemented, task too long | implement `expirationHandler`, keep tasks short |
| Extension can't access app data | missing app group entitlement | add `com.apple.security.application-groups` |
| Keyboard extension no network | `RequestsOpenAccess` false | set to true in extension Info.plist |
| Critical alert not working | missing entitlement | add `com.apple.developer.usernotifications.critical-alerts` |
| ATT prompt not showing | missing `NSUserTrackingUsageDescription`, wrong timing | add Info.plist key, request after onboarding |
| Privacy manifest missing | required reason APIs used without manifest | add `PrivacyInfo.xcprivacy`, declare APIs/data |
