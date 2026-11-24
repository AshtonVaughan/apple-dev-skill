# AppKit Reference (macOS)

AppKit is the native macOS UI framework — the oldest and most capable. SwiftUI covers most needs; AppKit for menu bar apps, complex document apps, custom windows, and AppKit-only APIs.

## 1. Application lifecycle

```swift
@main class AppDelegate: NSObject, NSApplicationDelegate {
  func applicationDidFinishLaunching(_ notification: Notification) { ... }
  func applicationShouldTerminateAfterLastWindowClosed(_ sender: NSApplication) -> Bool { true }
  func applicationSupportsSecureRestorableState(_ app: NSApplication) -> Bool { true }
  func application(_ app: NSApplication, open urls: [URL]) { ... }   // file-open / URL events
}
NSApplication.shared.setActivationPolicy(.regular)  // .accessory (menu bar agent), .prohibited
```

- `NSApplication.shared` — singleton; `run()` starts event loop; `terminate()`, `hide()`, `activate(ignoringOtherApps:)`.
- Activation policies: `.regular` (dock icon + menu bar), `.accessory` (no dock icon — menu bar apps, agents), `.prohibited` (background only).
- `Info.plist` `LSUIElement = YES` — launch as agent (no dock icon) at startup; equivalent to `.accessory`.
- Single instance: `applicationShouldHandleReopen` for dock-click behavior; document apps get `application(_:openFile:)`/`applicationShouldOpenUntitledFile`.
- `NSApp.mainMenu` — the app menu bar; build `NSMenu`/`NSMenuItem` hierarchy.

## 2. Windows

```swift
let window = NSWindow(contentRect: NSRect(x:0,y:0,width:800,height:600),
  styleMask: [.titled, .closable, .miniaturizable, .resizable, .fullSizeContentView],
  backing: .buffered, defer: false)
window.title = "My Window"
window.titlebarAppearsTransparent = true
window.titleVisibility = .hidden
window.isMovableByWindowBackground = true
window.center()
window.makeKeyAndOrderFront(nil)
window.level = .floating   // .normal, .floating, .modalPanel, .statusBar, .popUpMenu, .screenSaver
```

- `NSWindowController` — owns window + view controller; `showWindow(_:)`, `close()`, `windowWillClose` notification.
- Style masks: `.titled`, `.closable`, `.miniaturizable`, `.resizable`, `.fullScreen`, `.fullSizeContentView` (content under titlebar), `.unifiedTitleAndToolbar`, `.hudWindow`, `.nonactivatingPanel`.
- `NSPanel` — utility/floating panels: `.nonactivatingPanel`, `.utilityWindow`, `.hidesOnDeactivate`; used for inspectors, palettes, HUDs.
- Sheets: `window.beginSheet(sheetWindow, completionHandler:)` — attached modal; child window slides from titlebar.
- `NSWindow` frame autosave: `setFrameAutosaveName("main")` — remembers position/size.
- Multiple windows: each `NSWindow` has own `contentViewController`; SwiftUI `WindowGroup` handles this automatically.
- `window.sharingType = .readOnly/.none` — screen sharing privacy.
- `window.collectionBehavior` — `.fullScreenPrimary`, `.fullScreenAuxiliary`, `.canJoinAllSpaces`, `.moveToActiveSpace`, `.stationary`, `.ignoresCycle`.
- `NSWindow` `standardWindowButton(_:)` — access close/mini/zoom buttons for custom styling.
- `window.titlebarSeparatorStyle`, `toolbar` (`NSToolbar` + `NSToolbarItem` + `NSToolbarItemGroup`).
- Fullscreen: `window.toggleFullScreen(nil)`; `NSWindowDelegate` `windowWillUseStandardFrame`.
- Tabbed windows: `window.tabbingMode = .preferred`/`.disallowed`/`.automatic`.

## 3. Views & view controllers

```swift
class MyViewController: NSViewController {
  override func loadView() { view = NSView() }        // programmatic view creation
  override func viewDidLoad() { super.viewDidLoad(); setup() }  // ONCE
  override func viewWillAppear() { super.viewWillAppear(); }     // every appearance
  override func updateViewConstraints() { super.updateViewConstraints(); /* add constraints */ }
}
```

- `NSViewController` lifecycle mirrors UIKit: `loadView` → `viewDidLoad` → `viewWillAppear`/`viewDidAppear` → `viewWillDisappear`/`viewDidDisappear` → `viewDidUnload` (dead).
- Containment: `addChild(_:)`, `removeFromParent()`, `willMove(toParent:)`/`didMove(toParent:)` — same pattern as UIKit.
- `NSView` — `frame`, `bounds`, `center`, `autoresizingMask` (legacy), `wantsLayer`, `layer` (CALayer backing — always layer-backed on modern macOS).
- `NSStackView` — horizontal/vertical stack: `orientation`, `distribution` (`.fill`, `.fillEqually`, `.fillProportionally`, `.equalSpacing`, `.equalCentering`), `alignment` (`.leading`, `.centerY`, `.top`, `.firstBaseline` etc.), `spacing`, `edgeInsets`; `addArrangedSubview`, `insertArrangedSubview(_:at:)`.
- Auto Layout: same anchor API as UIKit (`NSLayoutAnchor`, `NSLayoutConstraint.activate`, visual format `constraints(withVisualFormat:)`), priorities 1-1000, `setContentHuggingPriority`/`setContentCompressionResistancePriority`.
- `NSLayoutGuide`/`NSLayoutAnchor` — layout guides for spacing without dummy views.
- Intrinsic content size: views report natural size (`intrinsicContentSize`); labels/buttons do automatically.
- `NSClipView`/`NSScrollView` — `documentView` (content), `contentView` (clip), scrollers; `NSScrollView.documentView` Auto Layout via `contentLayoutGuide`? — AppKit uses frame-based scroll views with `documentView` frame = content size.
- `NSTableView` — row/column table: `NSTableColumn`, `NSTableViewDataSource` (`numberOfRows`, `viewFor`/`dataCell`), `NSTableViewDelegate`; view-based cells via `NSTableCellView`; `outlineView` variant for disclosure trees (`NSOutlineView`); sorting via `sortDescriptors`.
- `NSCollectionView` — grid/icon/custom: `NSCollectionViewCompositionalLayout` (sections/groups/items like iOS), `NSCollectionViewFlowLayout` (legacy grid), diffable data source `NSCollectionViewDiffableDataSource`; `NSCollectionViewItem`.
- `NSOutlineView` — outline disclosure list for hierarchical data.
- `NSVisualEffectView` — vibrancy/blur: `material` (.headerView, .sheet, .windowBackground, .popover, .menu, .hudWindow, .fullScreenUI, .toolTip, .contentBackground, .underWindowBackground, .underPageBackground), `blendingMode` (.behindWindow/.withinWindow), `state` (.active/.inactive).
- `NSBox` — group box / separator box (bezel styles).
- `NSProgressIndicator` — `.bar`/`.spinning` styles, `startAnimation/stopAnimation`, `isIndeterminate`.
- `NSActivityIndicator`? — no, use NSProgressIndicator spinning.

## 4. Controls catalog

### Buttons & inputs
- `NSButton` — ` bezelStyle` (.rounded, .regularSquare, .disclosure, .helpButton, .smallSquare, .texturedRounded, .roundRect, .recessed, .recessedSquare), `controlSize` (.mini/.small/.regular/.large), `image`/`title`/`alternateTitle`/`imagePosition`, `keyEquivalent` (keyboard shortcut: "\r"=Return, "\u{1b}"=Esc, " "=Space, "\t"=Tab, "\u{7f}"=Delete, "\u{19}"=Pause, modifiers via `.keyEquivalentModifierMask` = [.command]/[.shift]/[.option]/[.control]); `NSButton.bezelColor`; `buttonType` (.momentaryPushIn, .toggle, .switch, .radio, .checkBox, .popUp, .pullDown, .inline).
- `NSPopUpButton` — dropdown menu button.
- `NSComboBox` — editable dropdown with completion.
- `NSSegmentedControl` — `.momentary`, `.selectOne`, `.selectAny`; segments with labels/images; `trackingMode`.
- `NSSlider` — horizontal/vertical/circular: `minValue`, `maxValue`, `doubleValue`, `isContinuous` (send every change vs on release), `tickMarkPosition`.
- `NSLevelIndicator` — capacity-style indicator (discrete/continuous).
- `NSColorWell` — color picker button.
- `NSDatePicker` — date/time picker (graphical/textual styles).
- `NSStepper` — increment/decrement.
- `NSSearchField` — search field with recent searches, `sendsSearchStringImmediately`, `sendsWholeSearchString`.
- `NSTextField` — single-line text: `stringValue`, `placeholderString`, `isEditable`, `isSelectable`, `bezelStyle`, `drawsBackground`, `backgroundColor`, `textColor`, `font`, `alignment`, `delegate` (`controlTextDidBeginEditing` etc.).
- `NSTextView` — rich text multi-line: `string`, `attributedString`, `isEditable`, `isRichText`, `isAutomaticQuoteSubstitutionEnabled`, `isAutomaticDashSubstitutionEnabled`, `isAutomaticTextReplacementEnabled`, `isAutomaticSpellingCorrectionEnabled`, `isContinuousSpellCheckingEnabled`, `allowsUndo`, `typingAttributes`, `textContainer`, `textStorage` (NSTextStorage for efficient editing), `layoutManager`, `textContainerInset`; used in `NSScrollView` for scrolling.
- `NSTokenField` — tokenized input (tags, recipients).
- `NSPathControl` — path display with popup menu.
- `NSBrowser` — column-view browser (Finder-style).
- `NSRuleEditor` — rule builder UI.
- `NSPredicateEditor` — predicate builder.

### Selection & pickers
- `NSPopUpButton`/`NSComboBox` — option selection.
- `NSSegmentedControl` — mode switching.
- `NSTableView`/`NSOutlineView`/`NSCollectionView` — list/grid selection (single/multiple via `allowsMultipleSelection`, `allowsEmptySelection`).
- `NSColorPanel`/`NSFontPanel`/`NSOpenPanel`/`NSSavePanel`/`NSPrintPanel` — system panels; run modally via `runModal()` or as sheets via `beginSheetModal`.

## 5. Menus & commands

```swift
let mainMenu = NSMenu()
let appMenuItem = NSMenuItem()
let appMenu = NSMenu()
appMenu.addItem(NSMenuItem(title: "About MyApp", action: #selector(about), keyEquivalent: ""))
appMenu.addItem(.separator())
appMenu.addItem(NSMenuItem(title: "Quit MyApp", action: #selector(NSApplication.terminate(_:)), keyEquivalent: "q"))
appMenuItem.submenu = appMenu
mainMenu.addItem(appMenuItem)
NSApp.mainMenu = mainMenu
```

- `NSMenu`/`NSMenuItem` — `title`, `keyEquivalent` (single char or special: "\r", "\u{1b}", " ", "\t", "\u{7f}", "\u{19}"), `keyEquivalentModifierMask` ([.command], [.option], [.control], [.shift] — combine), `image`, `submenu`, `isEnabled`, `state` (.on/.off/.mixed for checkmarks), `target`/`action` (nil target = responder chain).
- **Responder chain validation**: menu items with nil target traverse responder chain — `validateMenuItem(_:)` or `validateUserInterfaceItem(_:)` controls enabled state dynamically.
- `NSMenuItem.keyEquivalent` = "" means no shortcut.
- Context menus: `NSMenu.popUp(positioning:at:in:)` or `menu.popUpContextMenu(with:with:for:)`; or set `view.menu = NSMenu()` for right-click.
- `UIMenu`/`UIMenuBuilder` — SwiftUI/UIKit menu system on macOS via commands; AppKit uses NSMenu directly.
- Status bar: `NSStatusItem` via `NSStatusBar.system.statusItem(withLength:)` — `button.image`, `button.title`, `menu` for dropdown; `.variable`/`.fixed` length; accessory apps live here.
- Touch Bar: DEAD — removed in modern macOS; ignore.

## 6. Responder chain & events

- First responder: `window.firstResponder` — currently receiving key events; `makeFirstResponder(_:)`, `selectNextKeyView(_:)`.
- `NSResponder` chain: view → superview → viewController → window → windowController → app → appDelegate → nil.
- Override in NSView/NSViewController: `keyDown(with:)`, `keyUp(with:)`, `mouseDown/Up/Dragged/Moved/Entered/Exited`, `scrollWheel(with:)`, `pressureChange`, `rightMouseDown`, `otherMouseDown`.
- `NSEvent` — `type` (.keyDown, .keyUp, .leftMouseDown, .rightMouseDown, .otherMouseDown, .leftMouseDragged, .rightMouseDragged, .mouseMoved, .mouseEntered, .mouseExited, .scrollWheel, .pressure, .gesture, .magnify, .rotate, .swipe, .smartMagnify, .quickLook, .tabletPoint, .tabletProximity, .appKitDefined, .systemDefined, .applicationDefined, .periodic), `keyCode`, `characters`, `charactersIgnoringModifiers`, `modifierFlags` ([.command], [.shift], [.option], [.control], [.capsLock], [.numericPad], [.function], [.help], [.deviceIndependentFlagsMask]), `locationInWindow`, `deltaX/deltaY/deltaZ` (scroll), `phase`, `momentumPhase`, `pressure`, `rotation`, `magnification`.
- `NSEvent.addLocalMonitorForEvents(matching:handler:)` / `addGlobalMonitorForEvents` — intercept events before delivery.
- `NSTrackingArea` — mouse-entered/exited/moved without clicks: `NSTrackingArea(rect:options:owner:userInfo:)` with options `.mouseEnteredAndExited`, `.mouseMoved`, `.activeInKeyWindow`, `.activeAlways`, `.inVisibleRect`, `.cursorUpdate`; add via `view.addTrackingArea(_:)`; remove in `updateTrackingAreas`.
- Gesture recognizers: `NSClickGestureRecognizer`, `NSPanGestureRecognizer`, `NSMagnificationGestureRecognizer`, `NSRotationGestureRecognizer`, `NSPressGestureRecognizer` — add via `view.addGestureRecognizer(_:)`.

## 7. Drag & drop & pasteboard

### Drag & drop
```swift
// Source
let item = NSDraggingItem(pasteboardWriter: myObject)   // or NSPasteboardItem
item.draggingFrame = view.bounds
item.imageComponentsProvider = { ... }                  // drag image
view.beginDraggingSession(with: [item], event: event, source: self)

// Destination
view.registerForDraggedTypes([.string, .fileURL, NSPasteboard.PasteboardType("com.myapp.custom")])
// Implement NSDraggingDestination: draggingEntered/Updated/Exited/Perform/Conclude/Ended
```
- `NSDraggingSource`/`NSDraggingDestination` protocols; `NSDraggingSession` manages the drag.
- Pasteboard types: `.string`, `.URL`, `.fileURL`, `.tiff`, `.png`, `.color`, `.font`, `.html`, `.rtf`, `.pdf`, `.string`/`NSStringPboardType` legacy, custom via `NSPasteboard.PasteboardType("com.example.type")`.
- `NSPasteboardItem` — multiple representations of one dragged item.

### Pasteboard (copy/paste)
```swift
let pb = NSPasteboard.general
pb.clearContents()
pb.setString("text", forType: .string)
pb.setData(data, forType: .fileURL)
pb.writeObjects([url])                    // for NSURL/NSImage/NSString etc.
pb.readObjects(forClasses: [NSURL.self], options: nil)
pb.changeCount                        // increments on every write — poll for changes
pb.name                               // NSPasteboard.Name.general/.find/.ruler/.font
```
- `NSPasteboard.Name` — `.general`, `.find`, `.font`, `.ruler`, `.drag`.
- Promised files: `pb.setPropertyList(_:forType:)` or `NSFilePromiseProvider` for async file provision.
- Find pasteboard: `NSTextView.usesFindBar` + `.find` pasteboard for find/replace UI.

## 8. AppKit document architecture

- `NSDocument` — document-based app base class: `fileURL`, `fileName`, `displayName`, `isDocumentEdited`, `save(to:ofType:for:)`, `read(from:ofType:)`, `autosavesInPlace`, `keepsUndoManager`, `hasUndoManager`, `updateChangeCount`, `undoManager` (NSUndoManager), `windowControllers` array, `makeWindowControllers()`.
- `NSDocumentController` — shared doc controller: `openDocument(withContentsOf:display:completionHandler:)`, `newDocument`, `documentFor(window:)`, `documents` array.
- `NSPersistentDocument` — adds autosave + file coordination.
- `NSFileCoordinator` — coordinate file access across processes: `coordinate(readingItemAt:options:error:byAccessor:)`, `coordinate(writingItemAt:options:error:byAccessor:)`.
- Document windows: `NSWindowController` subclass per document; `windowControllers` on document.
- `NSDocument` autosave: `autosavesInPlace = true` + `autosavingFileType`.
- Versions: `NSFileVersion` for document versioning (macOS Versions feature).
- iCloud: `NSMetadataQuery` + ubiquity containers for iCloud Drive docs.

## 9. Notifications & workspace

- `UNUserNotificationCenter` — same as iOS: `requestAuthorization`, `UNMutableNotificationContent`, `UNTimeIntervalNotificationTrigger`, `UNCalendarNotificationTrigger`, `UNLocationNotificationTrigger`.
- `DistributedNotificationCenter` — cross-process notifications within same user session (sandboxed apps need `com.apple.security.temporary-exception` entitlement).
- `NSWorkspace.shared` — `notificationCenter` for workspace events (`didMount`, `didUnmount`, `didWake`, `screensDidSleep`, `didActivateApplication`, `willSleep`, `didLaunchApplication`), `open(url)`, `openFile`, `selectFile`, `urlForApplication`, `runningApplications`, `frontmostApplication`, `sharedWorkspace`.
- `NSRunningApplication` — `processIdentifier`, `localizedName`, `bundleIdentifier`, `isActive`, `activate()`, `terminate()`, `forceTerminate()`, `icon`.
- `NSWorkspace.shared.notificationCenter` notifications: `NSWorkspace.didWakeNotification`, `NSWorkspace.screensDidSleepNotification`, `NSWorkspace.willSleepNotification`, `NSWorkspace.sessionDidResignActiveNotification`, `NSWorkspace.sessionDidBecomeActiveNotification`, `NSWorkspace.didActivateApplicationNotification`, `NSWorkspace.didDeactivateApplicationNotification`, `NSWorkspace.didLaunchApplicationNotification`, `NSWorkspace.didTerminateApplicationNotification`, `NSWorkspace.didMountNotification`, `NSWorkspace.didUnmountNotification`, `NSWorkspace.didRenameVolumeNotification`.

## 10. Sandbox & security-scoped bookmarks

### Entitlements (com.apple.security.*)
```xml
<key>com.apple.security.app-sandbox</key><true/>
<key>com.apple.security.network.client</key><true/>      <!-- outbound network -->
<key>com.apple.security.network.server</key><true/>      <!-- inbound -->
<key>com.apple.security.files.user-selected.read-write</key><true/>  <!-- open/save panels -->
<key>com.apple.security.files.downloads.read-write</key><true/>      <!-- ~/Downloads -->
<key>com.apple.security.files.pictures.read-write</key><true/>       <!-- ~/Pictures -->
<key>com.apple.security.files.music.read-write</key><true/>
<key>com.apple.security.files.movies.read-write</key><true/>
<key>com.apple.security.files.bookmarks.app-scope</key><true/>       <!-- security-scoped bookmarks -->
<key>com.apple.security.files.bookmarks.document-scope</key><true/>
<key>com.apple.security.temporary-exception.files.absolute-path.read-write</key><array><string>/path</string></array>
<key>com.apple.security.device.camera</key><true/>
<key>com.apple.security.device.microphone</key><true/>
<key>com.apple.security.device.audio-input</key><true/>
<key>com.apple.security.personal-information.calendars</key><true/>
<key>com.apple.security.personal-information.addressbook</key><true/>
<key>com.apple.security.personal-information.photos-library</key><true/>
<key>com.apple.security.automation.apple-events</key><true/>          <!-- AppleScript control -->
<key>com.apple.security.scripting-targets</key><dict>...</dict>
<key>com.apple.security.print</key><true/>
<key>com.apple.security.smartcard</key><true/>
<key>com.apple.security.hardened-process.checked-allocations</key><true/>
```

### Security-scoped bookmarks
```swift
// Create bookmark for security-scoped access
let bookmark = try url.bookmarkData(options: [.withSecurityScope],
  includingResourceValuesForKeys: nil, relativeTo: nil)
// Later: resolve + start/stop access
var stale = false
let resolved = try URL(resolvingBookmarkData: bookmark, options: [],
  relativeTo: nil, bookmarkDataIsStale: &stale)
let ok = resolved.startAccessingSecurityScopedResource()
defer { resolved.stopAccessingSecurityScopedResource() }
```
- Required for sandboxed apps to persist access to user-selected files across launches.
- `NSOpenPanel`/`NSSavePanel` automatically provide security-scoped access during the panel session.

### Hardened runtime & notarization
- Hardened runtime entitlements: `com.apple.security.cs.allow-jit`, `allow-unsigned-executable-memory`, `allow-dyld-environment-variables`, `disable-library-validation`, `debugger`, `allow-dsstore`? — check `codesign -d --entitlements :-`.
- Notarization: `xcrun notarytool submit App.zip --keychain-profile PROFILE --wait` then `xcrun stapler staple App.app` — see `13-signing-distribution.md`.

## 11. Services, login items, helpers

- **Services**: `NSServices` in Info.plist or `NSRegisterServices` — expose app functionality to other apps via Services menu / right-click.
- **Login items**: `SMAppService` (ServiceManagement framework, macOS 13+): `SMAppService.mainApp.register()`, `SMAppService.loginItem(identifier:)`, `SMAppService.daemon(plistName:)`, `SMAppService.agent(plistName:)`; status via `.status` (`.enabled`, `.requiresApproval`, `.notFound`).
- **Helpers/XPC**: XPC services via `NSXPCConnection`/`NSXPCInterface`/`NSXPCListener`; launchd agents/daemons via `SMAppService.daemon`/`.agent` (modern replacement for deprecated `SMJobBless`).
- **Privileged helpers**: deprecated auth → `SMAppService.daemon` with `com.apple.developer.endpoint-security`? — endpoint security needs special entitlement from Apple.
- **Login items (legacy)**: `LSSharedFileList` — deprecated, use `SMAppService`.

## 12. Printing

- `NSPrintOperation` — `view.print(with:)` or `NSPrintOperation(view:printInfo:)`; `NSPrintInfo` for settings; `NSPrintPanel` for UI; `runModal` or sheet.
- `NSPrintOperationPdf`? — `NSPrintOperation` has `.pdf` attribute for generating PDFs.

## 13. macOS 27 (Golden Gate) specifics

- **Last macOS with Rosetta 2** — Intel apps still run via Rosetta; macOS 28 will drop it. Build universal or Apple-silicon-only; don't rely on Rosetta long-term.
- **Liquid Glass refinements**: materials adapt automatically; `.interactive()` glass for clickable elements; inactive window dimming automatic.
- **SwiftUI-first**: most new APIs arrive SwiftUI-first; AppKit gets them later or via representables.
- **Apple Intelligence**: `FoundationModels` framework works on macOS (on-device AFM 3 Core); `AppIntents` for Siri AI integration.
- **Window resizing**: Stage Manager, tiling — respect `window.collectionBehavior` and size constraints.
- **Menu bar minimal icons**: iPad/Mac menu bars show minimal icon set by default — design `NSStatusItem` accordingly.

## 14. SwiftUI ↔ AppKit interop

- `NSViewRepresentable` — wrap `NSView` in SwiftUI; `NSViewControllerRepresentable` — wrap `NSViewController`; `NSHostingController`/`NSHostingView` — embed SwiftUI in AppKit.
- `NSHostingView(rootView:)` — SwiftUI view as NSView; sizing via `sizingOptions` (`.intrinsicContentSize`, `.minSize`, `.maxSize`).
- `NSHostingController(rootView:)` — SwiftUI view as window content VC.
- `NSViewControllerRepresentable.Coordinator` — delegate/data source target.
- `NSViewRepresentable.sizeThatFits(_:nsView:proposedSize:)` — control sizing negotiation.

## 15. Common AppKit bugs → fixes

| Bug | Cause | Fix |
|---|---|---|
| View not appearing | `loadView` not called, view not added, frame zero | check `isViewLoaded`, add to superview, set frame/constraints |
| Constraints not working | `translatesAutoresizingMaskIntoConstraints` still true | set to false for programmatic constraints |
| Menu item disabled | `validateMenuItem` returning false or target nil not in responder chain | implement validation or set explicit target/action |
| Window not key | `makeKeyAndOrderFront` not called, `canBecomeKey` false | call `makeKeyAndOrderFront(nil)`, check `isReleasedWhenClosed` |
| Sheet not showing | `beginSheet` on wrong window or completion handler issue | ensure parent window is key, use `window.beginSheet` |
| Text view scrolling wrong | `NSTextView` not in `NSScrollView` | embed in scroll view, set `documentView` |
| Memory leak | retain cycle via delegate/block, `NSTimer` not invalidated | use `weak` delegates, invalidate timers in `deinit`/`viewWillDisappear` |
| Sandbox file access denied | missing entitlement or security-scoped bookmark | add entitlement, use bookmark for persistent access |
| Notarization fails | missing hardened runtime entitlements, invalid signature | check `codesign -d --entitlements`, `spctl -a -vv` |
| App Nap slowing timers | `NSProcessInfo.processInfo.automaticTerminationSupportEnabled` or App Nap | disable for critical apps, use `NSActivity` for background tasks |
| Multi-window state shared | singletons holding per-window state | use per-window controllers/models |

## 16. Performance

- Instruments: **Time Profiler**, **Allocations**, **Leaks**, **App Nap**? (Energy Log), **File Activity**, **Network**, **Hangs**, **Animation Hitches**.
- `NSView` drawing: override `draw(_:)` for custom drawing (called during display cycle); `setNeedsDisplay(_:)` to mark dirty; `displayIfNeeded()` to force; `layer`-backed views use `layerContentsRedrawPolicy`.
- `drawRect` vs layer-backed: layer-backed is default on modern macOS; `wantsLayer = true` for explicit layer control; `layerContentsPlacement`, `layerContentsRedrawPolicy`.
- `NSView` `layout()`/`layoutSubtreeIfNeeded()` — manual layout pass; `needsLayout = true` to mark dirty.
- `CATransaction` — batch layer changes: `CATransaction.begin()`, `setAnimationDuration`, `commit()`.
- `NSAnimationContext` — group animations: `runAnimationGroup { context in context.duration = 0.3; ... }`.
- `NSAnimation` — timer-based animations for views.
- `NSProgressIndicator` — use `.bar` for determinate, `.spinning` for indeterminate.
- App launch: `applicationDidFinishLaunching` should be fast; defer non-critical init; use `os_signpost` + `XCTest.measure` for launch metrics.
- Memory: `NSCache` for caches (auto-evicts under pressure), `NSMapTable` for weak-key dictionaries.

## 17. Decision cheat sheet

- Simple window + form → SwiftUI `WindowGroup` + `Form`.
- Menu bar app → SwiftUI `MenuBarExtra` or AppKit `NSStatusItem`.
- Document-based app → `NSDocument` + `NSDocumentController` (or SwiftUI `DocumentGroup`).
- Complex custom window chrome → AppKit `NSWindow` + custom `NSView` titlebar.
- Finder-style browser → `NSBrowser` or `NSOutlineView`.
- Rich text editor → `NSTextView` in `NSScrollView`.
- Inspector panel → `NSPanel` + `NSViewController` containment.
- Drag & drop → `NSDraggingSource`/`NSDraggingDestination`.
- Services/print panels → AppKit system panels.
- iCloud documents → `NSMetadataQuery` + ubiquity containers.
