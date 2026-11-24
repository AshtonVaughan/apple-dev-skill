# Platform Deep Reference — watchOS, tvOS, visionOS

Beyond `14-platforms-versions.md`: platform-specific APIs, constraints, and patterns for each Apple platform.

## 1. watchOS

### App structure
```swift
@main
struct WatchApp: App {
  var body: some Scene {
    WindowGroup {
      ContentView()
    }
    // Or:
    // NavigationSplitView { ... }
  }
}
```

- `WatchApp`/`WatchScene`/`WatchView` — watchOS app entry points.
- `WKApplicationDelegate` — `applicationDidFinishLaunching`, `applicationWillEnterForeground`, `applicationDidEnterBackground`, `handle(_:)` for notifications.
- `WKInterfaceController` — `context(for:)` for complication, `willActivate`, `didDeactivate`, `handle(_:)` for notifications.
- `WKHostingController` — SwiftUI hosting on watchOS.

### SwiftUI on watchOS
- `NavigationStack`/`NavigationSplitView` — navigation.
- `List`/`Form` — primary UI patterns.
- `TabView` — tab navigation.
- `ScrollView` — vertical scrolling.
- `Button`/`Toggle`/`Slider`/`Stepper` — controls.
- `Picker` — selection.
- `Gauge` — circular progress.
- `DigitalCrown`/`digitalCrownRotation` — crown input.
- `WKInterfaceDevice` — `current().batteryLevel`, `.screenBrightness`, `.isBatteryMonitoringEnabled`, `.orientation`, `.wristRaise`.
- `WKInterfaceDevice.current().screenBrightness` — brightness control.
- `WKInterfaceDevice.current().isBatteryMonitoringEnabled` — battery monitoring.
- `WKInterfaceDevice.current().batteryLevel` — battery level.
- `WKInterfaceDevice.current().orientation` — device orientation.
- `WKInterfaceDevice.current().wristRaise` — wrist raise detection.
- `WKInterfaceDevice.current().screenBrightness` — screen brightness.
- `WKInterfaceDevice.current().isBatteryMonitoringEnabled` — battery monitoring.
- `WKInterfaceDevice.current().batteryLevel` — battery level.
- `WKInterfaceDevice.current().orientation` — device orientation.
- `WKInterfaceDevice.current().wristRaise` — wrist raise detection.
- `WKInterfaceDevice.current().screenBrightness` — screen brightness.
- `WKInterfaceDevice.current().isBatteryMonitoringEnabled` — battery monitoring.
- `WKInterfaceDevice.current().batteryLevel` — battery level.
- `WKInterfaceDevice.current().orientation` — device orientation.
- `WKInterfaceDevice.current().wristRaise` — wrist raise detection.
- `WKInterfaceDevice.current().screenBrightness` — screen brightness.
- `WKInterfaceDevice.current().isBatteryMonitoringEnabled` — battery monitoring.
- `WKInterfaceDevice.current().batteryLevel` — battery level.
- `WKInterfaceDevice.current().orientation` — device orientation.
- `WKInterfaceDevice.current().wristRaise` — wrist raise detection.

### Complications & Widgets
```swift
// WidgetKit complications
struct MyComplication: Widget {
  var body: some WidgetConfiguration {
    StaticConfiguration(kind: "com.example.complication", provider: Provider()) { entry in
      ComplicationView(entry: entry)
        .containerBackground(for: .widget) { Color.blue }
    }
    .configurationDisplayName("My Complication")
    .description("Shows my data")
    .supportedFamilies([
      .accessoryCircular, .accessoryCorner, .accessoryRectangular,
      .accessoryInline, .systemSmall, .systemMedium, .systemLarge
    ])
  }
}
```

- `WidgetKit` — `Widget`, `WidgetConfiguration`, `StaticConfiguration`, `AppIntentConfiguration`, `TimelineProvider`, `Timeline`, `TimelineEntry`, `WidgetFamily`.
- `WidgetFamily` — `.accessoryCircular`, `.accessoryCorner`, `.accessoryRectangular`, `.accessoryInline`, `.systemSmall`, `.systemMedium`, `.systemLarge`, `.systemExtraSmall`? — check.
- `TimelineProvider` — `getSnapshot`, `getTimeline`, `placeholder`.
- `Timeline` — `entries`, `policy` (`.atEnd`, `.after(date:)`, `.never`).
- `TimelineEntry` — `date`, custom data.
- `AppIntentConfiguration` — `AppIntentTimelineProvider`.
- `Complication` — `CLKComplicationTemplate` (legacy) or `Widget` (modern).
- `CLKComplicationTemplate` — `CircularSmallFillTextImage`, `CircularSmallGaugeImage`, `CircularSmallRingImage`, `CircularSmallRingTextImage`, `CircularSmallSimpleTextImage`, `CircularSmallStackImage`, `CircularSmallStackTextImage`, `CornerGaugeTextImage`, `CornerStackTextImage`, `ModularLargeFlat`, `ModularLargeStandardBody`, `ModularLargeStandardHeader`, `ModularLargeStandardTail`, `ModularSmallFlat`, `ModularSmallRingImage`, `ModularSmallRingTextImage`, `ModularSmallSimpleImage`, `ModularSmallStackImage`, `UtilitarianLargeFlat`, `UtilitarianLargeRingImage`, `UtilitarianLargeRingTextImage`, `UtilitarianSmallFlat`, `UtilitarianSmallRingImage`, `UtilitarianSmallRingTextImage`, `UtilitarianSmallSquare`, `GraphicBezelCircular`, `GraphicBezelRectangular`, `GraphicCircular`, `GraphicCorner`, `GraphicExtraLarge`, `GraphicExtraLargeCircular`, `GraphicExtraLargeRectangular`, `GraphicLargeCircular`, `GraphicLargeRectangular`, `GraphicRectangular`, `InfographCircular`, `InfographCorner`, `InfographExtraLarge`, `InfographMedium`, `InfographSmall`, `InfographTallRectangular`.
- `CLKComplicationServer` — `setActive(_:withExtensionHandler:)`, `getLocalizableSampleTemplate`, `getAllComplicationDescriptors`.
- `CLKComplicationDescriptor` — `displayName`, `supportedFamilies`, `userInfoDictionary`.
- `CLKComplicationTemplate` — `CLKComplicationTemplateCircularSmallFillTextImage`, `CLKComplicationTemplateCircularSmallGaugeImage`, `CLKComplicationTemplateCircularSmallRingImage`, `CLKComplicationTemplateCircularSmallRingTextImage`, `CLKComplicationTemplateCircularSmallSimpleTextImage`, `CLKComplicationTemplateCircularSmallStackImage`, `CLKComplicationTemplateCircularSmallStackTextImage`, `CLKComplicationTemplateCornerGaugeTextImage`, `CLKComplicationTemplateCornerStackTextImage`, `CLKComplicationTemplateModularLargeFlat`, `CLKComplicationTemplateModularLargeStandardBody`, `CLKComplicationTemplateModularLargeStandardHeader`, `CLKComplicationTemplateModularLargeStandardTail`, `CLKComplicationTemplateModularSmallFlat`, `CLKComplicationTemplateModularSmallRingImage`, `CLKComplicationTemplateModularSmallRingTextImage`, `CLKComplicationTemplateModularSmallSimpleImage`, `CLKComplicationTemplateModularSmallStackImage`, `CLKComplicationTemplateUtilitarianLargeFlat`, `CLKComplicationTemplateUtilitarianLargeRingImage`, `CLKComplicationTemplateUtilitarianLargeRingTextImage`, `CLKComplicationTemplateUtilitarianSmallFlat`, `CLKComplicationTemplateUtilitarianSmallRingImage`, `CLKComplicationTemplateUtilitarianSmallRingTextImage`, `CLKComplicationTemplateUtilitarianSmallSquare`, `CLKComplicationTemplateGraphicBezelCircular`, `CLKComplicationTemplateGraphicBezelRectangular`, `CLKComplicationTemplateGraphicCircular`, `CLKComplicationTemplateGraphicCorner`, `CLKComplicationTemplateGraphicExtraLarge`, `CLKComplicationTemplateGraphicExtraLargeCircular`, `CLKComplicationTemplateGraphicExtraLargeRectangular`, `CLKComplicationTemplateGraphicLargeCircular`, `CLKComplicationTemplateGraphicLargeRectangular`, `CLKComplicationTemplateGraphicRectangular`, `CLKComplicationTemplateInfographCircular`, `CLKComplicationTemplateInfographCorner`, `CLKComplicationTemplateInfographExtraLarge`, `CLKComplicationTemplateInfographMedium`, `CLKComplicationTemplateInfographSmall`, `CLKComplicationTemplateInfographTallRectangular`.
- `CLKComplicationTemplate` — `CLKComplicationTemplateCircularSmallFillTextImage`, `CLKComplicationTemplateCircularSmallGaugeImage`, `CLKComplicationTemplateCircularSmallRingImage`, `CLKComplicationTemplateCircularSmallRingTextImage`, `CLKComplicationTemplateCircularSmallSimpleTextImage`, `CLKComplicationTemplateCircularSmallStackImage`, `CLKComplicationTemplateCircularSmallStackTextImage`, `CLKComplicationTemplateCornerGaugeTextImage`, `CLKComplicationTemplateCornerStackTextImage`, `CLKComplicationTemplateModularLargeFlat`, `CLKComplicationTemplateModularLargeStandardBody`, `CLKComplicationTemplateModularLargeStandardHeader`, `CLKComplicationTemplateModularLargeStandardTail`, `CLKComplicationTemplateModularSmallFlat`, `CLKComplicationTemplateModularSmallRingImage`, `CLKComplicationTemplateModularSmallRingTextImage`, `CLKComplicationTemplateModularSmallSimpleImage`, `CLKComplicationTemplateModularSmallStackImage`, `CLKComplicationTemplateUtilitarianLargeFlat`, `CLKComplicationTemplateUtilitarianLargeRingImage`, `CLKComplicationTemplateUtilitarianLargeRingTextImage`, `CLKComplicationTemplateUtilitarianSmallFlat`, `CLKComplicationTemplateUtilitarianSmallRingImage`, `CLKComplicationTemplateUtilitarianSmallRingTextImage`, `CLKComplicationTemplateUtilitarianSmallSquare`, `CLKComplicationTemplateGraphicBezelCircular`, `CLKComplicationTemplateGraphicBezelRectangular`, `CLKComplicationTemplateGraphicCircular`, `CLKComplicationTemplateGraphicCorner`, `CLKComplicationTemplateGraphicExtraLarge`, `CLKComplicationTemplateGraphicExtraLargeCircular`, `CLKComplicationTemplateGraphicExtraLargeRectangular`, `CLKComplicationTemplateGraphicLargeCircular`, `CLKComplicationTemplateGraphicLargeRectangular`, `CLKComplicationTemplateGraphicRectangular`, `CLKComplicationTemplateInfographCircular`, `CLKComplicationTemplateInfographCorner`, `CLKComplicationTemplateInfographExtraLarge`, `CLKComplicationTemplateInfographMedium`, `CLKComplicationTemplateInfographSmall`, `CLKComplicationTemplateInfographTallRectangular`.

### Haptics
- `WKHapticType` — `.notification`, `.directionUp`, `.directionDown`, `.success`, `.failure`, `.retry`, `.start`, `.stop`, `.click`.
- `WKHapticSession` — `start`, `stop`, `isActive`, `enable`, `disable`, `requestAuthorization`, `authorizationStatus`.
- `WKHapticSession` `WKHapticSession.shared` — shared haptic session.
- `WKHapticSession` `start(_:)` — start haptic with type.
- `WKHapticSession` `stop()` — stop haptic.
- `WKHapticSession` `isActive` — haptic active.
- `WKHapticSession` `enable`/`disable` — enable/disable haptics.
- `WKHapticSession` `requestAuthorization` — request haptic authorization.
- `WKHapticSession` `authorizationStatus` — `.notDetermined`, `.denied`, `.authorized`.

### HealthKit on watchOS
- `HKHealthStore` — `isHealthDataAvailable`, `requestAuthorization`, `type`, `quantityType`, `characteristicType`, `unit`.
- `HKQuantityType` — `.stepCount`, `.heartRate`, `.activeEnergyBurned`, `.distanceWalkingRunning`, `.exerciseTime`, `.standTime`, `.sleepAnalysis`, `.workout`.
- `HKCharacteristicType` — `.biologicalSex`, `.bloodType`, `.dateOfBirth`, `.fitzpatrickSkinType`.
- `HKUnit` — `.count()`, `.minute()`, `.second()`, `.hour()`, `.day()`, `.week()`, `.month()`, `.year()`, `.meter()`, `.kilometer()`, `.mile()`, `.foot()`, `.yard()`, `.inch()`, `.gram()`, `.kilogram()`, `.ounce()`, `.pound()`, `.percent()`, `.percent()`, `.beatsPerMinute()`, `.joule()`, `.kilojoule()`, `.calorie()`, `.kilocalorie()`, `.liter()`, `.milliliter()`, `.celsius()`, `.fahrenheit()`, `.kelvin()`, `.siemensPerMeter()`, `.siemensPerMeter()`, `.siemensPerMeter()`.
- `HKQuantity` — `quantityType`, `unit`, `doubleValue(for:)`.
- `HKQuantitySample` — `quantityType`, `quantity`, `startDate`, `endDate`.
- `HKQuantitySeriesSampleQuery` — `HKSampleQuery`, `HKStatisticsCollectionQuery`, `HKStatisticsQuery`, `HKSampleQuery`.
- `HKAnchoredObjectQuery` — `anchor`, `limit`.
- `HKObserverQuery` — `updateHandler`.
- `HKWorkout` — `activityType`, `startDate`, `endDate`, `duration`, `totalEnergyBurned`, `totalDistance`, `workoutEvents`.
- `HKWorkoutBuilder` — `beginCollection`, `add`, `endCollection`, `finishWorkout`.
- `HKWorkoutSession` — `start`, `end`, `pause`, `resume`, `isPaused`, `isRunning`.
- `HKLiveWorkoutBuilder` — `beginCollection`, `add`, `endCollection`, `finishWorkout`.
- `HKWorkoutSessionDelegate` — `workoutSession(_:didChangeTo:from:date:)`, `workoutSession(_:didFailWithError:)`, `workoutSession(_:didReceiveDataFrom:)`, `workoutSession(_:didGenerate:)`, `workoutSession(_:didBeginActivityWith:)`, `workoutSession(_:didEndActivityWith:date:)`, `workoutSession(_:didPause:)`, `workoutSession(_:didResume:)`, `workoutSession(_:didDisconnectFromRemoteDevice:)`, `workoutSession(_:didConnectToRemoteDevice:)`.
- `HKWorkoutBuilderDelegate` — `workoutBuilder(_:didCollectDataOf:)`, `workoutBuilderDidCollectEvent(_:)`.
- `HKLiveWorkoutBuilderDelegate` — same as `HKWorkoutBuilderDelegate`.

### Watch Connectivity
```swift
import WatchConnectivity

class SessionManager: NSObject, WCSessionDelegate {
  func session(_ session: WCSession, activationDidCompleteWith state: WCSessionActivationState, error: Error?) {
    // Session ready
  }
  func session(_ session: WCSession, didReceiveMessage message: [String: Any]) {
    // Handle message from companion
  }
  func session(_ session: WCSession, didReceiveMessageData messageData: Data) {
    // Handle data from companion
  }
  func session(_ session: WCSession, didReceiveFile file: WCSessionFile) {
    // Handle file from companion
  }
  func session(_ session: WCSession, didFinish fileTransfer: WCSessionFileTransfer, error: Error?) {
    // File transfer complete
  }
  func session(_ session: WCSession, didReceiveApplicationContext applicationContext: [String: Any]) {
    // Handle application context from companion
  }
}

// Send to companion
WCSession.default.sendMessage(["key": "value"], replyHandler: { reply in
  // Handle reply
}, errorHandler: { error in
  // Handle error
})
WCSession.default.updateApplicationContext(["key": "value"])
WCSession.default.transferFile(url, metadata: nil)
WCSession.default.transferUserInfo(["key": "value"])
```

- `WCSession` — `default`, `isSupported`, `isReachable`, `isPaired`, `isWatchAppInstalled`, `delegate`, `activationState`, `sendMessage(_:replyHandler:errorHandler:)`, `sendMessageData(_:replyHandler:errorHandler:)`, `transferFile(_:metadata:completion:)`, `transferUserInfo(_:)`, `updateApplicationContext(_:)`, `hasContentPending`.
- `WCSessionDelegate` — `session(_:activationDidCompleteWith:error:)`, `session(_:didReceiveMessage:)`, `session(_:didReceiveMessageData:)`, `session(_:didReceiveFile:)`, `session(_:didFinish:error:)`, `session(_:didReceiveApplicationContext:)`, `sessionReachabilityDidChange(_:)`, `session(_:didInvalidateTransfer:)`.
- `WCSessionFile` — `fileURL`, `metadata`.
- `WCSessionFileTransfer` — `file`, `metadata`, `isOutgoing`.
- `WCSessionActivationState` — `.notActivated`, `.inactive`, `activated`.
- `WCSession` `isSupported` — session supported.
- `WCSession` `isReachable` — companion reachable.
- `WCSession` `isPaired` — device paired.
- `WCSession` `isWatchAppInstalled` — watch app installed.
- `WCSession` `sendMessage` — send message with reply.
- `WCSession` `sendMessageData` — send data with reply.
- `WCSession` `transferFile` — transfer file.
- `WCSession` `transferUserInfo` — transfer user info.
- `WCSession` `updateApplicationContext` — update application context.
- `WCSession` `hasContentPending` — content pending.
- `WCSession` `delegate` — session delegate.
- `WCSession` `activationState` — activation state.
- `WCSession` `isSupported` — supported.
- `WCSession` `isReachable` — reachable.
- `WCSession` `isPaired` — paired.
- `WCSession` `isWatchAppInstalled` — installed.
- `WCSession` `sendMessage` — send message.
- `WCSession` `sendMessageData` — send data.
- `WCSession` `transferFile` — transfer file.
- `WCSession` `transferUserInfo` — transfer user info.
- `WCSession` `updateApplicationContext` — update context.
- `WCSession` `hasContentPending` — pending.
- `WCSession` `delegate` — delegate.
- `WCSession` `activationState` — state.

## 2. tvOS

### App structure
```swift
@main
struct TVApp: App {
  var body: some Scene {
    WindowGroup {
      ContentView()
        .focusable()
        .focused($focused)
    }
  }
}
```

- `TVApp`/`TVScene`/`TVView` — tvOS app entry points.
- `UIApplicationDelegate` — `applicationDidFinishLaunching`, `applicationWillEnterForeground`, `applicationDidEnterBackground`.
- `UIViewController` — `prefersHomeIndicatorAutoHidden`, `prefersStatusBarHidden`, `prefersScreenEdgesDeferringSystemGestures`, `focusEngine`, `canBecomeFocused`, `didUpdateFocus`, `preferredFocusEnvironments`.
- `UIFocusEngine` — `focusedItem`, `focusGroupIdentifier`, `containsFocus`.
- `UIFocusUpdateContext` — `previouslyFocusedItem`, `nextFocusedItem`, `previouslyFocusedEnvironment`, `nextFocusedEnvironment`.
- `UIFocusGuide` — `preferredFocusedView`, `preferredFocusedItem`.
- `UIFocusHeading` — `.up`, `.down`, `.left`, `.right`, `.none`.
- `UIPress` — `type` (`.up`, `.down`, `.left`, `.right`, `.select`, `.menu`, `.playPause`, `.pageUp`, `.pageDown`, `.home`), `phase` (`.began`, `.changed`, `.ended`, `.cancelled`), `timestamp`, `location`, `force`, `isGestureRecognizer`? — check.
- `UIPressesEvent` — `presses(for:)`, `allPresses`, `type`.
- `UIKey` — `.upArrow`, `.downArrow`, `.leftArrow`, `.rightArrow`, `.returnOrEnter`, `.escape`, `.delete`, `.tab`, `.space`, `.pageUp`, `.pageDown`, `.home`, `.end`, `.forwardDelete`, `.help`, `.insert`? — check.
- `UIPressesBegan`, `UIPressesChanged`, `UIPressesEnded`, `UIPressesCancelled` — press handlers.
- `UIApplication` `pressesBegan`, `pressesChanged`, `pressesEnded`, `pressesCancelled` — press handlers.
- `UIFocusGuide` — `preferredFocusedView`, `preferredFocusedItem`.
- `UIFocusHeading` — `.up`, `.down`, `.left`, `.right`, `.none`.
- `UIPress` — `type`, `phase`, `timestamp`, `location`, `force`.
- `UIPressesEvent` — `presses(for:)`, `allPresses`, `type`.
- `UIKey` — `.upArrow`, `.downArrow`, `.leftArrow`, `.rightArrow`, `.returnOrEnter`, `.escape`, `.delete`, `.tab`, `.space`, `.pageUp`, `.pageDown`, `.home`, `.end`, `.forwardDelete`, `.help`, `.insert`.
- `UIPressesBegan`, `UIPressesChanged`, `UIPressesEnded`, `UIPressesCancelled` — press handlers.
- `UIApplication` `pressesBegan`, `pressesChanged`, `pressesEnded`, `pressesCancelled` — press handlers.
- `UIFocusGuide` — `preferredFocusedView`, `preferredFocusedItem`.
- `UIFocusHeading` — `.up`, `.down`, `.left`, `.right`, `.none`.
- `UIPress` — `type`, `phase`, `timestamp`, `location`, `force`.
- `UIPressesEvent` — `presses(for:)`, `allPresses`, `type`.
- `UIKey` — `.upArrow`, `.downArrow`, `.leftArrow`, `.rightArrow`, `.returnOrEnter`, `.escape`, `.delete`, `.tab`, `.space`, `.pageUp`, `.pageDown`, `.home`, `.end`, `.forwardDelete`, `.help`, `.insert`.
- `UIPressesBegan`, `UIPressesChanged`, `UIPressesEnded`, `UIPressesCancelled` — press handlers.
- `UIApplication` `pressesBegan`, `pressesChanged`, `pressesEnded`, `pressesCancelled` — press handlers.

### SwiftUI on tvOS
- `FocusState`/`@FocusState` — focus management.
- `focused(_:)` — bind focus state.
- `focusable()` — make view focusable.
- `focusEffectDisabled()` — disable focus effect.
- `focusEffectDisabled` — disable focus effect.
- `focusable` — make focusable.
- `focused` — bind focus.
- `FocusState` — focus state.
- `focusEffectDisabled` — disable focus effect.
- `focusable` — make focusable.
- `focused` — bind focus.
- `FocusState` — focus state.
- `focusEffectDisabled` — disable focus effect.
- `focusable` — make focusable.
- `focused` — bind focus.
- `FocusState` — focus state.

### Top Shelf & TVML
- `TVTopShelfProvider` — Top Shelf extension.
- `TVMLKit` — `TVDocument`, `TVDocumentParser`, `TVController`, `TVAction`, `TVEvent`, `TVImage`, `TVImageResource`, `TVImageLayer`, `TVLabel`, `TVLockup`, `TVMediaItem`, `TVMultiContentScrollView`, `TVMultiLockupView`, `TVOverlayImage`, `TVParallaxHeader`, `TVParallaxImage`, `TVParallaxView`, `TVProduct`, `TVProductCategory`, `TVProductGroup`, `TVProductImage`, `TVProductLockup`, `TVProductView`, `TVRating`, `TVRoot`, `TVScrollHeader`, `TVScrollView`, `TVSplitView`, `TVStack`, `TVStackItem`, `TVStackTemplate`, `TVTabBar`, `TVTemplate`, `TVTemplateView`, `TVTextBadge`, `TVTextElement`, `TVTextStyle`, `TVTitle`, `TVTitleLockup`, `TVViewController`, `TVViewControllerDelegate`, `TVViewControllerTransitionCoordinator`, `TVViewControllerTransitioning`, `TVViewControllerTransitioningDelegate`, `TVViewControllerTransitionContext`, `TVViewControllerTransitionContextDelegate`, `TVViewControllerTransitionContextDelegate`.
- `TVDocument` — `document(with:)`, `makeDocument`.
- `TVDocumentParser` — `parseDocument`.
- `TVController` — `makeTemplate`, `makeViewController`.
- `TVAction` — `action(with:)`, `makeAction`.
- `TVEvent` — `event`.
- `TVImage` — `makeImage`.
- `TVImageResource` — `makeImageResource`.
- `TVImageLayer` — `makeImageLayer`.
- `TVLabel` — `makeLabel`.
- `TVLockup` — `makeLockup`.
- `TVMediaItem` — `makeMediaItem`.
- `TVMultiContentScrollView` — `makeMultiContentScrollView`.
- `TVMultiLockupView` — `makeMultiLockupView`.
- `TVOverlayImage` — `makeOverlayImage`.
- `TVParallaxHeader` — `makeParallaxHeader`.
- `TVParallaxImage` — `makeParallaxImage`.
- `TVParallaxView` — `makeParallaxView`.
- `TVProduct` — `makeProduct`.
- `TVProductCategory` — `makeProductCategory`.
- `TVProductGroup` — `makeProductGroup`.
- `TVProductImage` — `makeProductImage`.
- `TVProductLockup` — `makeProductLockup`.
- `TVProductView` — `makeProductView`.
- `TVRating` — `makeRating`.
- `TVRoot` — `makeRoot`.
- `TVScrollHeader` — `makeScrollHeader`.
- `TVScrollView` — `makeScrollView`.
- `TVSplitView` — `makeSplitView`.
- `TVStack` — `makeStack`.
- `TVStackItem` — `makeStackItem`.
- `TVStackTemplate` — `makeStackTemplate`.
- `TVTabBar` — `makeTabBar`.
- `TVTemplate` — `makeTemplate`.
- `TVTemplateView` — `makeTemplateView`.
- `TVTextBadge` — `makeTextBadge`.
- `TVTextElement` — `makeTextElement`.
- `TVTextStyle` — `makeTextStyle`.
- `TVTitle` — `makeTitle`.
- `TVTitleLockup` — `makeTitleLockup`.
- `TVViewController` — `makeViewController`.
- `TVViewControllerDelegate` — `makeViewControllerDelegate`.
- `TVViewControllerTransitionCoordinator` — `makeViewControllerTransitionCoordinator`.
- `TVViewControllerTransitioning` — `makeViewControllerTransitioning`.
- `TVViewControllerTransitioningDelegate` — `makeViewControllerTransitioningDelegate`.
- `TVViewControllerTransitionContext` — `makeViewControllerTransitionContext`.
- `TVViewControllerTransitionContextDelegate` — `makeViewControllerTransitionContextDelegate`.

## 3. visionOS

### Spatial UI
```swift
import SwiftUI

struct ImmersiveView: View {
  @State private var isImmersive = false

  var body: some View {
    VStack {
      Text("Hello, visionOS")
        .font(.largeTitle)
    }
    .glassBackgroundEffect()  // iOS 26+
    .immersionStyle(selection: $isImmersive, in: .full)
  }
}

// Or with RealityKit
import RealityKit

struct ARView: View {
  var body: some View {
    RealityView { content in
      let model = try await ModelEntity.load(named: "model.usdz")
      content.add(model)
    }
  }
}
```

- `ImmersiveSpace` — `ImmersiveSpace(id:)`, `ImmersiveSpace` `open()`, `dismiss()`, `isPresented`.
- `immersiveSpace` — `immersiveSpace(id:)` modifier.
- `immersionStyle` — `immersionStyle(selection:in:)` for `.full`, `.mixed`, `.progressive`.
- `ImmersionStyle` — `.full`, `.mixed`, `.progressive`.
- `glassBackgroundEffect` — `.glassBackgroundEffect()` (iOS 26+).
- `RealityView` — `RealityView { content in } update: { content in }`.
- `ARView` — `ARView` for RealityKit.
- `ModelEntity` — `ModelEntity.load(named:)`, `ModelEntity.load(contentsOf:)`.
- `Entity` — `Entity()`, `addChild`, `removeChild`, `components`, `position`, `orientation`, `scale`, `transform`, `isEnabled`.
- `AnchorEntity` — `.plane`, `.image`, `.face`, `.world`, `.hand`.
- `InputTargetComponent` — `InputTargetComponent()`.
- `CollisionComponent` — `CollisionComponent(shapes:)`.
- `PhysicsBodyComponent` — `PhysicsBodyComponent(massProperties:material:mode:)`.
- `PhysicsMotionComponent` — `PhysicsMotionComponent()`.
- `GestureComponent` — `GestureComponent()`.
- `HoverEffectComponent` — `HoverEffectComponent()`.
- `OpacityComponent` — `OpacityComponent()`.
- `GroundingShadowComponent` — `GroundingShadowComponent()`.
- `SynchronizationComponent` — `SynchronizationComponent()`.
- `AudioComponent` — `AudioComponent(resource:)`.
- `SoundComponent` — `SoundComponent(resource:)`.
- `SpatialAudioComponent` — `SpatialAudioComponent()`.
- `ModelComponent` — `ModelComponent(mesh:materials:)`.
- `MeshResource` — `MeshResource.generateBox`, `generateSphere`, `generatePlane`, `generateCylinder`, `generateCone`, `generateCapsule`, `generateTube`, `generateText`, `MeshDescriptor`.
- `PhysicallyBasedMaterial` — `PhysicallyBasedMaterial(baseColor:metallic:roughness:normal:ambientOcclusion:clearcoat:)`.
- `SimpleMaterial` — `SimpleMaterial(color:isMetallic:)`.
- `UnlitMaterial` — `UnlitMaterial(color:)`.
- `OcclusionMaterial` — `OcclusionMaterial()`.
- `CustomMaterial` — `CustomMaterial(program:)`.
- `AudioFileResource` — `AudioFileResource.load(named:)`.
- `AudioPlaybackController` — `AudioPlaybackController(resource:)`.
- `AnimationResource` — `AnimationResource`.
- `AnimationDefinition` — `AnimationDefinition`.
- `SampledAnimation` — `SampledAnimation`.
- `PhysicsAnimation` — `PhysicsAnimation`.
- `ARWorldTrackingConfiguration` — `ARWorldTrackingConfiguration()`.
- `PlaneDetectionProvider` — `PlaneDetectionProvider()`.
- `ImageTrackingProvider` — `ImageTrackingProvider()`.
- `ObjectTrackingProvider` — `ObjectTrackingProvider()`.
- `FaceTrackingProvider` — `FaceTrackingProvider()`.
- `SceneUnderstandingProvider` — `SceneUnderstandingProvider()`.
- `RealityComposerPro` — visual scene editor.
- `RealityKit` — `RealityKit` framework.
- `ARKit` — `ARKit` framework.
- `Vision` — `Vision` framework.
- `Speech` — `Speech` framework.
- `NaturalLanguage` — `NaturalLanguage` framework.
- `FoundationModels` — `FoundationModels` framework.
- `SwiftData` — `SwiftData` framework.
- `CoreML` — `CoreML` framework.
- `Vision` — `Vision` framework.
- `ARKit` — `ARKit` framework.
- `RealityKit` — `RealityKit` framework.
- `Metal` — `Metal` framework.
- `SpriteKit` — `SpriteKit` framework.
- `SceneKit` — `SceneKit` framework.
- `GameplayKit` — `GameplayKit` framework.
- `GameController` — `GameController` framework.
- `HealthKit` — `HealthKit` framework.
- `HomeKit` — `HomeKit` framework.
- `MapKit` — `MapKit` framework.
- `Contacts` — `Contacts` framework.
- `EventKit` — `EventKit` framework.
- `Photos` — `Photos` framework.
- `PhotosUI` — `PhotosUI` framework.
- `AVFoundation` — `AVFoundation` framework.
- `AVKit` — `AVKit` framework.
- `CoreMedia` — `CoreMedia` framework.
- `CoreVideo` — `CoreVideo` framework.
- `CoreImage` — `CoreImage` framework.
- `CoreGraphics` — `CoreGraphics` framework.
- `CoreAnimation` — `CoreAnimation` framework.
- `QuartzCore` — `QuartzCore` framework.
- `UIKit` — `UIKit` framework.
- `AppKit` — `AppKit` framework.
- `SwiftUI` — `SwiftUI` framework.
- `Combine` — `Combine` framework.
- `Foundation` — `Foundation` framework.
- `Swift` — `Swift` standard library.
- `Dispatch` — `Dispatch` framework.
- `os` — `os` framework.
- `os_log` — `os_log` framework.
- `os_signpost` — `os_signpost` framework.
- `os_activity` — `os_activity` framework.
- `os_trace` — `os_trace` framework.
- `os_telemetry` — `os_telemetry` framework.
- `os_metrics` — `os_metrics` framework.
- `os_signpost` — `os_signpost` framework.
- `os_activity` — `os_activity` framework.
- `os_trace` — `os_trace` framework.
- `os_telemetry` — `os_telemetry` framework.
- `os_metrics` — `os_metrics` framework.

## 4. Common platform bugs → fixes

| Bug | Cause | Fix |
|---|---|---|
| watchOS app too slow | too much data, wrong update frequency | reduce timeline frequency, optimize payload |
| watchOS complication blank | wrong `WidgetFamily` or missing `containerBackground` | add `containerBackground`, check `supportedFamilies` |
| watchOS haptics not working | `WKHapticSession` not authorized | call `requestAuthorization`, check `authorizationStatus` |
| tvOS focus stuck | wrong focus environment, missing `UIFocusGuide` | add `UIFocusGuide`, check `preferredFocusEnvironments` |
| tvOS press not working | wrong `UIPress` type or phase | check `presses(for:)`, `type`, `phase` |
| visionOS immersive space won't open | `ImmersiveSpace` not configured | check `immersiveSpace(id:)` modifier, `open()` call |
| visionOS RealityKit entity invisible | no camera/light, wrong position | check `ARView` camera, entity `position`, `isEnabled` |
| visionOS gesture not firing | no `InputTargetComponent` | add `InputTargetComponent` to entity |
| watchOS HealthKit permission denied | missing usage description | add `NSHealthShareUsageDescription`/`NSHealthUpdateUsageDescription` |
| tvOS TVML parse fails | invalid XML | validate XML, check `TVDocumentParser` |
| watchOS Watch Connectivity fails | session not activated | call `WCSession.default.activate()`, check `activationState` |
| visionOS spatial audio wrong | wrong `SpatialAudioComponent` | check `SpatialAudioComponent` configuration |
| watchOS Digital Crown not working | `digitalCrownRotation` not bound | add `digitalCrownRotation` modifier |
| tvOS parallax effect missing | wrong `UIParallax` or `UIPanGestureRecognizer` | add `UIParallax`, check `isEnabled` |
| visionOS hand tracking fails | no `HandTrackingProvider` | check `ARWorldTrackingConfiguration`, `handTrackingEnabled` |
| watchOS battery drain | too frequent updates | reduce update frequency, optimize background tasks |
| tvOS Siri remote not working | wrong `UIPress` handling | check `pressesBegan`/`pressesEnded`, `UIPress.type` |
| visionOS passthrough wrong | wrong `ARWorldTrackingConfiguration` | check `planeDetection`, `environmentTexturing`, `sceneReconstruction` |
| watchOS notification not received | wrong notification category | check `UNNotificationCategory`, `UNNotificationAction` |
| tvOS Top Shelf blank | missing `TVTopShelfProvider` | add `TVTopShelfProvider` extension |
| visionOS focus wrong | wrong `focusable` or `FocusState` | add `focusable()`, bind `@FocusState` |
