# UIKit Reference (iOS/iPadOS)

UIKit is the imperative UI framework — still required for many APIs SwiftUI lacks, and the basis of all iOS apps. Modern approach: SwiftUI for most UI, UIKit for gaps + representables.

## 1. Architecture essentials

- **MVC-ish**: `UIViewController` owns a `UIView` hierarchy; view controllers manage lifecycle + user interaction; models are separate.
- **Scene-based lifecycle** (iOS 13+): `UISceneDelegate`/`UIWindowScene` replaces single `UIApplicationDelegate` window management. Apps have scenes; each scene has its own window(s), state, and restoration.
- App entry: `@main App` (SwiftUI) generates scene config, OR `UIApplicationMain` + `AppDelegate` + `SceneDelegate` in `Info.plist` (`UIApplicationSceneManifest`).

## 2. View controller lifecycle (order matters)

```
init(coder:) / init(nibName:bundle:) / init()
  → loadView()          // create view hierarchy programmatically (rarely override)
  → viewDidLoad()       // ONCE per VC instance; view loaded. Setup UI, NOT frame-dependent work
  → viewWillAppear()    // before visible; may be called repeatedly (every navigation back)
  → viewDidAppear()     // visible; start animations, tracking, notifications
  → viewWillLayoutSubviews() / viewDidLayoutSubviews()  // EVERY layout pass — expensive here
  → viewWillTransition(to:with:)  // size class/orientation change
  → viewWillDisappear() / viewDidDisappear()
  → viewDidUnload()     // NEVER called since iOS 6 — dead
  → deinit
```

- `viewDidLoad` = one-time setup. `viewWillAppear`/`viewDidAppear` = every appearance. Don't put one-time setup in appear — it runs on every back-navigation.
- `isMovingToParent`, `isMovingFromParent` in `willMove(toParent:)`/`didMove(toParent:)` for containment.
- Memory warnings: `didReceiveMemoryWarning()` — release caches, NOT the view (system may purge view under memory pressure in old iOS; modern keeps it).
- `setNeedsLayout()`/`layoutIfNeeded()` vs `setNeedsUpdateConstraints()`/`updateConstraintsIfNeeded()` — two-phase layout: constraints pass then layout pass.

## 3. Views & Auto Layout

### Frame vs constraints
- `frame`/`bounds`/`center` — absolute layout; breaks on rotation/size changes. Use for simple fixed cases or inside `layoutSubviews`.
- **Auto Layout** — constraint-based; the standard:

```swift
view.translatesAutoresizingMaskIntoConstraints = false   // REQUIRED for programmatic constraints
NSLayoutConstraint.activate([
  label.topAnchor.constraint(equalTo: view.safeAreaLayoutGuide.topAnchor, constant: 16),
  label.leadingAnchor.constraint(equalTo: view.leadingAnchor, constant: 16),
  button.topAnchor.constraint(equalTo: label.bottomAnchor, constant: 8),
  button.trailingAnchor.constraint(equalTo: view.trailingAnchor, constant: -16),
])
// Visual format (legacy but still works):
// NSLayoutConstraint.constraints(withVisualFormat: "H:|-16-[label]-8-[button]-16-|", ...)
```
- Anchors: `topAnchor/bottomAnchor/leadingAnchor/trailingAnchor/centerXAnchor/centerYAnchor/widthAnchor/heightAnchor`, `safeAreaLayoutGuide`, `layoutMarginsGuide`, `readableContentGuide`.
- Attributes: `.equal`, `.greaterThanOrEqual`, `.lessThanOrEqual`; multiplier for ratios: `widthAnchor.constraint(equalTo: heightAnchor, multiplier: 16.0/9.0)`.
- **Compression resistance** (490 default) vs **content hugging** (250 default) priorities 1-1000: higher wins when space is tight. `setContentHuggingPriority(.required, for: .horizontal)` to prevent stretching.
- `UIStackView` — horizontal/vertical stack with distribution (`.fill`, `.fillEqually`, `.equalSpacing`, `.equalCentering`) and alignment; auto-manages arranged subviews' constraints. THE fast way to lay out rows/columns.
- `UIScrollView` + Auto Layout: pin content to `contentLayoutGuide`, frame to `frameLayoutGuide`; or use `contentSize` manually.
- Ambiguous layout = console warnings "Unable to simultaneously satisfy constraints"; unsatisfiable = broken layout. Debug with View Debugger (Debug > View Debugger) — 3D hierarchy + constraint inspection.

### Safe area & traits
- `view.safeAreaInsets`, `safeAreaLayoutGuide` — avoid notch/home indicator/status bar.
- `traitCollection` — `userInterfaceStyle` (dark/light), `horizontalSizeClass`/`verticalSizeClass` (compact/regular), `displayScale`, `legibilityWeight`, `prefersCrossFadeTransitions`, accessibility traits.
- Override: `override var traitCollection` or `registerForTraitChanges([UITraitUserInterfaceStyle.self])` (iOS 17+) for change callbacks.
- `UIViewPropertyList`? no. `preservesSuperviewLayoutMargins`, `layoutMargins` for spacing.

## 4. View controller containment & presentation

### Navigation
- `UINavigationController` — stack of VCs; `pushViewController(_:animated:)`, `popViewController`, `popToViewController`, `setViewControllers`; `navigationItem` (title, leftBarButtonItem, rightBarButtonItems, prompt, largeTitleDisplayMode); `navigationBar` appearance via `UINavigationBarAppearance`.
- `UITabBarController` — `viewControllers` array; each child gets `UITabBarItem` (title/image/selectedImage/badgeValue); `tabBar` standard or custom via delegate.
- `UISplitViewController` — master/detail (iPad); `preferredDisplayMode` (.automatic/.oneBesideSecondary/.twoBesideSecondary/.twoOverSecondary); `showDetailViewController`; collapse/expand behavior; `UISplitViewControllerDelegate` for collapse customization.
- Custom container: `addChild(_:)`, `child.addConstraints` or child's view pinned to parent's view, `didMove(toParent:)` — required sequence:
  ```swift
  addChild(child); view.addSubview(child.view)
  child.view.frame = view.bounds // or constraints
  child.didMove(toParent: self)
  // removal: willMove(toParent: nil); view.removeFromSuperview(); removeFromParent()
  ```

### Modal presentation
- `present(_:animated:completion:)` / `dismiss(animated:)`.
- `modalPresentationStyle`: `.fullScreen`, `.pageSheet` (default iOS 13+ card), `.formSheet` (iPad centered), `.currentContext`, `.overFullScreen`, `.overCurrentContext`, `.popover`, `.automatic`, `.none`.
- `.pageSheet`/`.formSheet` get `pageSheetPrefersGrabberVisible`, `prefersEdgeAttachedInCompactHeight`.
- Custom transitions: `UIViewControllerTransitioningDelegate` + `UIViewControllerAnimatedTransitioning` — present/dismiss custom animations.
- Adaptive: same VC can present differently on iPhone vs iPad via `adaptivePresentationStyle` / trait overrides.
- `isModalInPresentation = true` — prevent swipe-to-dismiss.

## 5. Controls & views catalog

### Display
- `UILabel` — `text`, `font`, `textColor`, `numberOfLines` (0=unlimited), `lineBreakMode`, `attributedText`, `adjustsFontSizeToFitWidth`, `minimumScaleFactor`.
- `UIImageView` — `image`, `contentMode` (`.scaleAspectFit/.scaleAspectFill/.scaleToFill/.center`), `tintColor` for template images, `highlightedImage`.
- `UITextView` — rich text via `attributedText`, `isEditable`, `isScrollEnabled`, `textContainerInset`, `inputAccessoryView`, delegates.
- `WKWebView` — modern web view (WKWebViewConfiguration: preferences, websiteDataStore, userContentController for JS bridge). NOT UIWebView (dead).
- `UIVisualEffectView` — blur/vibrancy: `UIBlurEffect(style: .systemMaterial/.systemThinMaterial/.systemUltraThinMaterial/.dark/.light/.regular/.prominent)`, `.effect`, `UIVibrancyEffect` for labels on blur.
- `UIActivityIndicatorView` — `.style = .medium/.large`, `startAnimating/stopAnimating`, `hidesWhenStopped`.
- `UIProgressView` — `.default`/`.bar` styles, `setProgress(_:animated:)`.
- `UISwitch`, `UISlider` (min/max/value/isContinuous), `UISegmentedControl`, `UIStepper`, `UIDatePicker` (styles: `.wheels`, `.compact`, `.inline`, `.automatic`), `UIPageControl`.

### Input
- `UITextField` — `text`, `placeholder`, `borderStyle`, `keyboardType`, `returnKeyType`, `isSecureTextEntry`, `leftView/rightView` (accessory views), `delegate` for editing events, `inputAccessoryView` (keyboard toolbar), `textContentType` (emailAddress, password, oneTimeCode — enables AutoFill).
- `UITextView` — multi-line, `typingAttributes`, `selectedRange`.
- `UISearchBar` — `searchBarStyle`, `scopeButtonTitles`, `showsCancelButton`, delegate; or `UISearchController` for integrated search.
- `UIPickerView` — single/multi component selection.
- `UIDatePicker` in `.compact` = button popover.

### Buttons & bar items
- `UIButton` — `UIButton.Configuration` (iOS 15+): `.plain/.gray/.tinted/.filled/.bordered/.borderedProminent/.borderless`, cornerStyle, image+title+subtitle, `configurationUpdateHandler` for state changes. Legacy `setTitle/setImage` still works.
- `UIBarButtonItem` — system items (`.done`, `.cancel`, `.add`, `.action`, `.bookmarks`, `.search`, `.refresh`, `.flexibleSpace`, `.fixedSpace`), custom views.
- `UIContextualAction`/`UISwipeActionsConfiguration` — table row swipe actions.

### Collection views
- `UITableView` — single-column list; `UITableViewDataSource` (`numberOfRowsInSection`, `cellForRowAt`), `UITableViewDelegate` (didSelect, heightForRow, trailingSwipeActions); styles `.plain/.grouped/.insetGrouped`; `UITableViewDiffableDataSource` for modern declarative updates.
- `UICollectionView` — grid/custom layouts; `UICollectionViewCompositionalLayout` (sections with items/groups, orthogonal scrolling), `UICollectionViewFlowLayout` (simple grid), custom `UICollectionViewLayout` subclass; diffable data source same as table.
- Self-sizing cells: `UITableView.automaticDimension` + `estimatedRowHeight`; collection view self-sizing via preferredLayoutAttributesFitting.
- `UICollectionViewDiffableDataSource` — snapshot-based: `apply(NSDiffableDataSourceSnapshot)`; sections+items with `Hashable` IDs; animated insert/delete/move automatically. THE modern way.
- Cell reuse: `dequeueReusableCell(withIdentifier:)` — always reuse; configure in `cellForRow` not in `init`.

## 6. Scroll views

- `UIScrollView` — `contentSize`, `contentOffset`, `contentInset`, `contentInsetAdjustmentBehavior` (`.automatic` respects safe area), `scrollIndicatorInsets`, `isPagingEnabled`, `bounces`, `alwaysBounceVertical`, `showsVerticalScrollIndicator`, `contentLayoutGuide`/`frameLayoutGuide` for Auto Layout.
- `contentOffset.y` negative = pulled down (refresh); `contentInset.top` adjusted by safe area automatically.
- Zooming: `viewForZooming`, `minimumZoomScale`/`maximumZoomScale`, `zoom(to:animated:)`.
- `UIRefreshControl` — `refreshControl` on scroll view/table/collection; `.valueChanged` action.
- Nested scroll views: inner gets touches first; `delaysContentTouches`, `canCancelContentTouches` tuning.

## 7. Alerts, action sheets, activity

```swift
let alert = UIAlertController(title: "Delete?", message: "...", preferredStyle: .alert)
alert.addAction(UIAlertAction(title: "Cancel", style: .cancel))
alert.addAction(UIAlertAction(title: "Delete", style: .destructive) { _ in ... })
present(alert, animated: true)

// Action sheet (iPad requires popover source!)
let sheet = UIAlertController(title: nil, message: nil, preferredStyle: .actionSheet)
sheet.popoverPresentationController?.sourceView = button   // REQUIRED on iPad
sheet.popoverPresentationController?.sourceRect = button.bounds
```
- Styles: `.alert` (centered dialog), `.actionSheet` (bottom on iPhone, popover on iPad).
- Text fields in alerts: `alert.addTextField { $0.placeholder = "Name" }`.
- `UIActivityViewController(activityItems: [url, text])` — share sheet; `excludedActivityTypes` to hide options; popover source required on iPad.
- `UIDocumentPickerViewController` — open/import documents; `UIDocumentPickerMode.import/.open/.exportToService/.moveToService/.create`.

## 8. Gestures & touches

- `UIGestureRecognizer` subclasses: `UITapGestureRecognizer` (numberOfTapsRequired, numberOfTouchesRequired), `UILongPressGestureRecognizer` (minimumPressDuration), `UIPanGestureRecognizer` (translation(in:), velocity(in:), state .began/.changed/.ended/.cancelled/.failed/.possible), `UIPinchGestureRecognizer` (scale), `UIRotationGestureRecognizer` (rotation), `UISwipeGestureRecognizer` (direction), `UIScreenEdgePanGestureRecognizer` (edges).
- Add: `view.addGestureRecognizer(recognizer)`; delegate for simultaneous recognition (`gestureRecognizer(_:shouldRecognizeSimultaneouslyWith:)`, `shouldRequireFailureOf`).
- `UITouch`/`UIEvent`/`UIResponder` — low-level touch handling: `touchesBegan/Moved/Ended/Cancelled`, `hitTest(_:with:)`/`point(inside:with:)` for custom hit areas.
- Responder chain: touch → gesture recognizers → view → superview → VC → window → app → app delegate. `next` property walks it; override `touchesX` or add gesture recognizers (preferred).

## 9. Appearance, traits & dark mode

- `traitCollection.userInterfaceStyle` — `.unspecified/.light/.dark`; override per-view: `override var traitCollection` or `traitOverrides.userInterfaceStyle = .dark`.
- Dynamic colors: `UIColor { trait in trait.userInterfaceStyle == .dark ? .white : .black }` or asset catalog colors with Any/Dark appearances — ALWAYS use for custom colors.
- `UIColor.systemBackground`, `.label`, `.secondaryLabel`, `.systemBlue` etc. — semantic colors adapt automatically. `UIColor(named:)` for asset catalog.
- `UIStatusBarStyle` — `.default`/`.lightContent` via `preferredStatusBarStyle` on VC.
- Fonts: `UIFont.preferredFont(forTextStyle: .body)` + `UIFontMetrics` for scaling with Dynamic Type.
- SF Symbols: `UIImage(systemName: "gear")`, `UIImage.SymbolConfiguration(pointSize:weight:)`, `.withTintColor`, `.applyingSymbolConfiguration`.
- `UIAppearance` proxy (legacy): `UINavigationBar.appearance().tintColor = ...` — global styling; prefer per-instance or `UIBarAppearance` for bars.
- `UIView` semantic content attributes: `semanticContentAttribute = .forceLeftToRight/.forceRightToLeft` for RTL control.

## 10. Scene lifecycle & app states

```swift
func scene(_ scene: UIScene, willConnectTo session: UISceneSession, options connectionOptions: UIScene.ConnectionOptions)
// connectionOptions: urlContexts (deep links), userActivities (Handoff), notificationResponse, shortcutItem

func sceneDidBecomeActive(_ scene: UIScene)      // foreground, interactive
func sceneWillResignActive(_ scene: UIScene)     // losing focus (control center, phone call)
func sceneWillEnterForeground(_ scene: UIScene)  // returning from background
func sceneDidEnterBackground(_ scene: UIScene)   // entering background — save state NOW
func scene(_ scene: UIScene, openURLContexts:)   // URL opened
func scene(_ scene: UIScene, continue userActivity: NSUserActivity)  // Handoff/universal link
```
- App states: not running → inactive → active → inactive → background → suspended. Background time is LIMITED (~30s typical); use `BGTaskScheduler` for deferred work.
- `UIApplication.shared.applicationState` — `.active/.inactive/.background`.
- `scenePhase` in SwiftUI Environment mirrors this.
- Background modes (`UIBackgroundModes` in Info.plist): `audio`, `location`, `fetch`, `remote-notification`, `voip`, `bluetooth-central/peripheral`, `external-accessory`, `processing`, `newsstand` (dead). Only declare what you use — App Review checks.

## 11. Background execution

- `BGTaskScheduler.shared.register(forTaskWithIdentifier:using:launchHandler:)` — register in `didFinishLaunching` BEFORE app finishes launching; schedule via `BGProcessingTaskRequest` (requires external power for some) or `BGAppRefreshTaskRequest`.
- Task expiration: system calls expiration handler — finish quickly or get killed.
- `UIApplication.beginBackgroundTask(expirationHandler:)` — short grace period (~30s) for finishing work on backgrounding.
- Silent push: `content-available: 1` payload → `didReceiveRemoteNotification:fetchCompletionHandler:` → call completion handler.
- `BGContinuedProcessingTask` (iOS 26+) — longer-running background tasks with progress reporting for things like ML processing, large imports.
- Significant location change, region monitoring (`CLCircularRegion`/`CLBeaconRegion`), Core Bluetooth state restoration, audio session — ways to stay alive in background.

## 12. Photos, camera, documents

- `PHPickerViewController` — modern photo picker, out-of-process, limited access friendly: `PHPickerConfiguration(photoLibrary: .shared())`, `selectionLimit`, `filter = .images/.videos/.livePhotos/.any(of:)`; delegate `didFinishPicking` results `PHPickerResult` → `NSItemProvider.loadObject(ofClass:)` or `loadFileRepresentation`.
- `UIImagePickerController` — camera or legacy photo library; `sourceType = .camera/.photoLibrary/.savedPhotosAlbum`; requires `NSCameraUsageDescription` for camera; limited on modern iOS.
- `AVCaptureSession` — camera pipeline: inputs (`AVCaptureDeviceInput`) → `AVCaptureVideoPreviewLayer` → outputs (`AVCapturePhotoOutput`, `AVCaptureVideoDataOutput`, `AVCaptureMovieFileOutput`, `AVCaptureAudioDataOutput`, `AVCaptureMetadataOutput`); configure on serial queue; `session.startRunning()` is BLOCKING — call on background queue.
- `UIDocumentPickerViewController` — Files app integration.
- `QLPreviewController` — Quick Look previews for file URLs (dataSource/delegate).
- `UIActivityViewController` — share sheet (see §7).

## 13. Web views

- `WKWebView` — full browser engine; `WKWebViewConfiguration`: `websiteDataStore` (`.default()/.nonPersistent()`), `userContentController` (add `WKScriptMessageHandler` for JS→native bridge, `WKUserScript` for injected JS), `preferences` (`javaScriptEnabled` default true, `isElementFullscreenEnabled`); navigation delegate `WKNavigationDelegate` (decidePolicyFor, didFinish, didFail); `WKUIDelegate` for alerts/JS dialogs; `WKWebsiteDataStore` for cookies/storage; `WKContentRuleList` for ad blocking.
- `SFSafariViewController` — in-app Safari with shared cookies/autofill (use for OAuth-ish web flows, NOT for custom JS bridge); `SFSafariViewController.Configuration` for reader mode, bar tint.
- `ASWebAuthenticationSession` — THE way to do OAuth/web auth: system auth sheet, shares Safari cookies, returns callback URL; `prefersEphemeralWebBrowserSession` for private mode; requires `ASWebAuthenticationSession` + callback scheme in entitlements? — needs `webcredentials`? no, just URL scheme handling.

## 14. iPad-specific & multitasking

- `UISplitViewController` — master-detail; collapse to single column on iPhone; `preferredPrimaryColumnWidthFraction`.
- Size classes: compact width = iPhone portrait / narrow split; regular = iPad / wide split. Use trait overrides + `UITraitCollection` to adapt layout.
- `UIRequiresFullScreen` — opt out of Slide Over/Split View multitasking (avoid unless necessary — App Review may reject).
- Drag & drop: `UIDragInteraction`/`UIDropInteraction` delegates, `UIDragItem`, `NSItemProvider` — works on iPad + iPhone.
- Pointer/pencil: `UIPencilInteraction`, hover (`UIHoverGestureRecognizer`), indirect pointer (`UIPointerInteraction`), `traitCollection.pointerType`.
- Keyboard shortcuts on iPad: `UIKeyCommand` — `keyCommandWithInput` or `UIKeyCommand(input:modifierFlags:action:)`; discoverability via `UIMenuBuilder`/`UIMenuSystem`.
- `UIViewController` `override var keyCommands`.

## 15. Performance

- **Main thread only** for UI; heavy work → `DispatchQueue.global().async` or `Task.detached`; hop back with `DispatchQueue.main.async` or `@MainActor`.
- Instruments: **Time Profiler** (CPU hotspots), **Allocations** (memory growth), **Leaks**, **Animation Hitches** (dropped frames), **Core Animation** (offscreen rendering, blends), **os_signpost** intervals.
- Cell reuse, image downsampling (don't decode full-res into tiny thumbnail — `UIImage.preparingForDisplay()` or ImageIO thumbnail), prefetching (`UICollectionViewDataSourcePrefetching`), `CADisplayLink` for custom animation loops (invalidate when done).
- Auto Layout cost: complex constraint hierarchies are expensive; flatten where possible, use `UIStackView`, avoid constraint churn in scroll.
- `layer.shouldRasterize` for complex static content; `layer.drawsAsynchronously` for expensive drawing.
- `preconditionFailure`/`assert` — keep asserts in debug only.
- App launch: `didFinishLaunching` should be fast; defer non-critical init; measure with `os_signpost` + XCTest `measure(metrics:)`.

## 16. Debugging UIKit

- **View Debugger** (Debug > View Debugger): 3D exploded hierarchy, select view → see constraints, frames, properties. THE tool for layout issues.
- **Memory Graph Debugger**: shows retain cycles as object graphs.
- Constraint conflicts: console "Unable to simultaneously satisfy constraints" → identify ambiguous views, check priorities.
- `po [[UIWindow keyWindow] recursiveDescription]` in LLDB for view hierarchy dump.
- `UIView.alertForUnsatisfiableConstraints` — breakpoint on this symbol catches layout conflicts at runtime.
- Symbolic breakpoints: `UIViewAlertForUnsatisfiableConstraints`, `NSInternalInconsistencyException`, `objc_exception_throw`.
- `lldb` po: `po self.view.recursiveDescription`, `po self.constraints`, `image lookup -vn <symbol>`.
- Runtime issues (purple warnings in Xcode): Main Thread Checker, Thread Sanitizer, Address Sanitizer, Undefined Behavior Sanitizer — enable in scheme diagnostics.

## 17. Modern UIKit (iOS 17-27 additions)

- `UIContentUnavailableConfiguration` — empty states (`contentUnavailableConfiguration = UIContentUnavailableConfiguration.search()` etc.).
- `UISheetPresentationController` — `.medium()/.large()` detents, `.largestUndimmed`, grabber, custom detents via `UISheetPresentationController.Detent.custom`.
- `UIUpdateLink` (iOS 18+) — display-synced updates replacing CADisplayLink for SwiftUI-adjacent work.
- `UIButton.Configuration` — fully customizable buttons without custom views.
- `UIMainMenuSystem`/`UIMenuBuilder` — keyboard shortcut + menu building.
- `UISearchTab`/`UISearchTabPlacement` — iOS 26+ search in tab bars.
- `UIListContentConfiguration`/`UIBackgroundConfiguration` — cell content config API.
- `UIHoverGestureRecognizer`, `UIPointerStyle` — pointer/pencil support.
- `UIViewController` `updateProperties()`/`viewIsAppearing(_:)` — iOS 17+ lifecycle additions.
- `UITraitCollection` `accessibilityReduceMotion` etc. — honor all accessibility traits.
- `UIKeyCommand` `.discoverabilityTitle` for keyboard shortcut HUD.
- `UIActivityItemsConfiguration` — async share items.
- `UIGlassEffect` (iOS 26+) — Liquid Glass effect views; automatic adoption in bars/toolbars.
- `UIUpdateLink` — frame-synced updates.
- `UINavigationBarAppearance`/`UIToolbarAppearance`/`UITabBarAppearance` — consistent bar styling across iOS versions.
- `UIScrollView` `topEdgeEffect`/`bottomEdgeEffect` — scroll edge effects customization.
- `UIView` `tintColorDidChange`, `override func tintColorDidChange()`.
- `UIWindowScene` `effectiveGeometry` — window geometry including safe areas.
- `UIApplicationDelegate` `application(_:configurationForConnecting:options:)` for scene setup.

## 18. Quick decision table

| Need | Use |
|---|---|
| Simple form/settings UI | SwiftUI `Form` |
| Complex custom layout | SwiftUI stacks or UIKit+Auto Layout |
| Long scrollable list | `UITableView`/`UICollectionView` (UIKit perf) or SwiftUI `List`/`LazyVStack` |
| Camera capture | AVFoundation `AVCaptureSession` (UIKit preview layer or SwiftUI representable) |
| Web content | `WKWebView` (SwiftUI representable) or `SFSafariViewController` |
| OAuth | `ASWebAuthenticationSession` |
| Share sheet | `UIActivityViewController` (SwiftUI `ShareLink` wraps it) |
| Document picker | `UIDocumentPickerViewController` |
| Custom transition | `UIViewControllerTransitioningDelegate` |
| Drag & drop | `UIDragInteraction`/`UIDropInteraction` or SwiftUI `.draggable`/`.dropDestination` |
| Rich text editing | `UITextView` (SwiftUI `TextEditor` is limited) |
| Map | `MKMapView` (UIKit) or SwiftUI `Map` (simpler cases) |
| Notifications | `UNUserNotificationCenter` (not UIKit-specific) |
