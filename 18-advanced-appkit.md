# Advanced AppKit — Internals & Expert APIs

Deep-dive beyond `05-appkit-macos.md`: responder chain, view drawing, TextKit on macOS, bindings/KVO, custom window chrome, sheets, drag/drop, pasteboard, services, printing, undo, documents, state restoration, accessibility, Catalyst.

## 1. Responder chain deep

```
NSView → superview → NSViewController → NSWindow → NSWindowController → NSApplication → NSAppDelegate → nil
```

- First responder: `window.firstResponder` — receives key events; `makeFirstResponder(_:)`, `selectNextKeyView(_:)`, `selectPreviousKeyView(_:)`.
- `nextResponder` — walk the chain; override in NSView/NSViewController.
- Key events: `keyDown(with:)`, `keyUp(with:)`, `flagsChanged(with:)`, `interpretKeyEvents(_:)` — call `interpretKeyEvents` for standard key bindings (Cmd+C/V/X/Z/A).
- `NSResponder` methods: `mouseDown/Up/Dragged/Entered/Exited/Moved`, `rightMouseDown/Up/Dragged`, `otherMouseDown/Up/Dragged`, `scrollWheel(with:)`, `pressureChange`, `quickLook`, `smartMagnify`, `rotate`, `magnify`, `swipe`, `beginGesture`/`endGesture`.
- `NSResponder` `wantsForwardedScrollEvents` — receive scroll events when not first responder.
- `NSResponder` `tryToPerform(_:with:)` — attempt action on responder chain.
- `NSResponder` `validRequestor(forSendType:returnType:)` — pasteboard type negotiation.
- `NSResponder` `performKeyEquivalent(with:)` — key equivalent handling (Cmd+key).
- `NSResponder` `doCommand(by:)` — selector-based commands (cut:, copy:, paste:, undo:, redo:).
- `NSApplication` `sendAction(_:to:from:)` — send action through responder chain.
- Menu validation: `validateMenuItem(_:)`/`validateUserInterfaceItem(_:)` — called for each menu item; return `false` to disable.
- `NSMenuItem` `isEnabled` — manual override; `target`/`action` nil = responder chain.

## 2. View drawing & layer-backed views

### draw(_:) vs layer-backed
- `draw(_ dirtyRect:)` — called during display cycle; use for custom drawing with `NSGraphicsContext`/`CGContext`.
- `wantsLayer = true` — layer-backed view; `layer` property available; `layerContentsRedrawPolicy` controls when contents redraw.
- `layerContentsRedrawPolicy`: `.never`, `.onSetNeedsDisplay`, `.beforeViewResize`, `.crossfade`.
- `layerContentsPlacement` — `.center`, `.top`, `.bottom`, `.left`, `.right`, `.topLeft` etc.
- `layerContentsGravity` — `kCAGravityResize`, `kCAGravityResizeAspect`, `kCAGravityCenter` etc.
- `needsDisplay = true` / `setNeedsDisplay(_:)` — mark dirty; `displayIfNeeded()` — force display.
- `layout()`/`layoutSubtreeIfNeeded()` — manual layout pass; `needsLayout = true`.

### NSView drawing APIs
```swift
override func draw(_ dirtyRect: NSRect) {
  super.draw(dirtyRect)
  NSColor.red.setFill()
  dirtyRect.fill()
  // or
  let path = NSBezierPath(roundedRect: bounds, xRadius: 8, yRadius: 8)
  path.lineWidth = 2
  NSColor.blue.setStroke()
  path.stroke()
}
```

- `NSGraphicsContext.current` — current drawing context; `NSGraphicsContext.saveGraphicsState()`/`restoreGraphicsState()`.
- `NSBezierPath` — vector paths: `move(to:)`, `line(to:)`, `curve(to:controlPoint1:controlPoint2:)`, `close()`, `fill()`, `stroke()`, `appendRect`, `appendOval`, `appendArc`.
- `NSColor`/`NSGradient`/`NSShadow` — colors, gradients, shadows.
- `NSString.draw(at:withAttributes:)`/`NSAttributedString.draw(in:)` — text drawing.
- `NSImage.draw(in:from:operation:fraction:)` — image drawing.
- `NSAffineTransform` — transformations: `translate`, `rotate`, `scale`, `concat`, `invert`.
- `CATransaction` — batch layer changes: `CATransaction.begin()`, `setAnimationDuration`, `setDisableActions`, `commit()`.
- `NSAnimationContext` — group animations: `runAnimationGroup { context in context.duration = 0.3; context.allowsImplicitAnimation = true; ... }`.
- `NSAnimation` — timer-based animations for views: `NSAnimationContext.runAnimationGroup`.

## 3. Text system on macOS

- `NSTextView` — rich text editor: `string`, `attributedString`, `isEditable`, `isRichText`, `isAutomaticQuoteSubstitutionEnabled`, `isAutomaticDashSubstitutionEnabled`, `isAutomaticTextReplacementEnabled`, `isAutomaticSpellingCorrectionEnabled`, `isContinuousSpellCheckingEnabled`, `allowsUndo`, `typingAttributes`, `textContainer`, `textStorage` (NSTextStorage for efficient editing), `layoutManager`, `textContainerInset`.
- `NSTextStorage` — text storage (NSMutableAttributedString subclass) with layout manager attachment.
- `NSLayoutManager` — TextKit 1 layout: `usedRect(for:)`, `boundingRect(forGlyphRange:in:)`, `glyphRange(forBoundingRect:in:)`, `characterRange(forGlyphRange:actualGlyphRange:)`.
- `NSTextContainer` — text container: `size`, `lineFragmentPadding`, `exclusionPaths`, `maximumNumberOfLines`, `lineBreakMode`.
- `NSTextLayoutManager` — TextKit 2 layout (macOS 12+): `NSTextLayoutFragment`, `NSTextLine`.
- `NSTextAttachment` — inline images/objects.
- `NSTextList` — ordered/unordered lists.
- `NSParagraphStyle` — `NSMutableParagraphStyle`: `alignment`, `lineSpacing`, `paragraphSpacing`, `lineBreakMode`, `hyphenationFactor`, `tabStops`, `defaultTabInterval`, `allowsDefaultTighteningForTruncation`, `usesDefaultHyphenation`, `lineHeightMultiple`.
- `NSAttributedString` attributes: `.font`, `.foregroundColor`, `.backgroundColor`, `.kern`, `.ligature`, `.baselineOffset`, `.underlineStyle`, `.strikethroughStyle`, `.attachment`, `.link`, `.shadow`, `.strokeColor`, `.strokeWidth`, `.obliqueness`, `.expansion`, `.paragraphStyle`, `.toolTip`, `.cursor`, `.textAlternatives`? — check.
- `NSMutableAttributedString` — efficient editing.
- `NSDataDetector` — links, phone numbers, dates detection.
- `NSTextCheckingResult` — detection results.
- `NSTextView` in `NSScrollView` — `documentView` = textView, `hasVerticalScroller`, `autohidesScrollers`.
- `NSTextView` `isRichText` — rich text editing; `isEditable` — editing; `isSelectable` — selection only.
- `NSTextView` `allowsUndo` — undo/redo support.
- `NSTextView` `isAutomaticTextReplacementEnabled` — autocorrect.
- `NSTextView` `isAutomaticSpellingCorrectionEnabled` — spell check.
- `NSTextView` `isContinuousSpellCheckingEnabled` — continuous spell check.
- `NSTextView` `isGrammarCheckingEnabled` — grammar check.
- `NSTextView` `isAutomaticQuoteSubstitutionEnabled` — smart quotes.
- `NSTextView` `isAutomaticDashSubstitutionEnabled` — smart dashes.
- `NSTextView` `isAutomaticLinkDetectionEnabled` — link detection.
- `NSTextView` `isAutomaticDataDetectionEnabled` — data detection.
- `NSTextView` `isAutomaticTextCompletionEnabled` — text completion.
- `NSTextView` `isAutomaticSpellingCorrectionEnabled` — spell check.
- `NSTextView` `isContinuousSpellCheckingEnabled` — continuous spell check.
- `NSTextView` `isGrammarCheckingEnabled` — grammar check.
- `NSTextView` `isAutomaticQuoteSubstitutionEnabled` — smart quotes.
- `NSTextView` `isAutomaticDashSubstitutionEnabled` — smart dashes.
- `NSTextView` `isAutomaticLinkDetectionEnabled` — link detection.
- `NSTextView` `isAutomaticDataDetectionEnabled` — data detection.
- `NSTextView` `isAutomaticTextCompletionEnabled` — text completion.

## 4. Bindings & KVO

### Key-Value Observing (KVO)
```swift
// Manual KVO
obj.addObserver(self, forKeyPath: "name", options: [.new, .old], context: nil)
// Must implement:
override func observeValue(forKeyPath keyPath: String?, of object: Any?, change: [NSKeyValueChangeKey : Any]?, context: UnsafeMutableRawPointer?) {
  // Handle change
}
// Cleanup:
obj.removeObserver(self, forKeyPath: "name")
```

- `NSKeyValueObserving` protocol — `observe(_:options:changeHandler:)` returns `NSKeyValueObservation` token.
- `NSKeyValueObservation` — `invalidate()` to stop observing.
- KVO works on `@objc dynamic` properties, `NSObject` subclasses, `NSManagedObject`.
- KVO on Swift classes: needs `@objc dynamic var` or `NSObject` subclass.

### Key-Value Binding (Cocoa bindings)
```swift
// Bind view to model
textField.bind("value", to: model, withKeyPath: "name", options: [.continuouslyUpdatesValue])
slider.bind("value", to: model, withKeyPath: "volume", options: nil)
// Unbind:
textField.unbind("value")
```

- `NSObject.bind(_:to:withKeyPath:options:)` — bind control to model property.
- `NSObject.unbind(_:)` — remove binding.
- `NSBindingName` — `value`, `title`, `enabled`, `hidden`, `font`, `textColor`, `backgroundColor`, `toolTip`, `image`, `alternateImage`, `selectedCell`, `selectedObjects`, `arrangedObjects`, `content`, `sortDescriptors`, `selectionIndexes`.
- `NSKeyValueBindingOptions` — `.continuouslyUpdatesValue`, `.allowsNullValue`, `.nullPlaceholder`, `.notNullPlaceholder`, `.validatesImmediately`, `.handlesContentAsCompoundValue`, `.conditionallySetsHidden`, `.observesImmediately`, `.valueValidation`.
- Bindings use KVO under the hood — works on `@objc dynamic` properties.
- **SwiftUI doesn't use bindings** — uses `@Binding`/`@State`/`@Observable` instead.

## 5. Custom window chrome

```swift
class CustomWindow: NSWindow {
  override var canBecomeKey: Bool { true }
  override var canBecomeMain: Bool { true }
  override func constrainFrameRect(_ frameRect: NSRect, to screen: NSScreen?) -> NSRect {
    // Custom frame constraints
    var rect = frameRect
    if let screen { rect.size.width = min(rect.size.width, screen.visibleFrame.width * 0.8) }
    return rect
  }
}

// Borderless window
let window = NSWindow(contentRect: .zero, styleMask: [.borderless, .resizable], backing: .buffered, defer: false)
window.isMovableByWindowBackground = true
window.hasShadow = true
window.backgroundColor = .clear
window.isOpaque = false
```

- `NSWindow` `styleMask`: `.titled`, `.closable`, `.miniaturizable`, `.resizable`, `.fullScreen`, `.fullSizeContentView`, `.unifiedTitleAndToolbar`, `.hudWindow`, `.nonactivatingPanel`, `.utilityWindow`, `.docModalWindow`, `.borderless`.
- `NSPanel` — `nonactivatingPanel`, `utilityWindow`, `hidesOnDeactivate`, `becomesKeyOnlyIfNeeded`.
- `window.titlebarAppearsTransparent`, `window.titleVisibility`, `window.titlebarSeparatorStyle`.
- `window.standardWindowButton(_:)` — `.closeButton`, `.miniaturizeButton`, `.zoomButton`, `.toolbarButton`, `.documentIconButton`, `.documentActionsButton`.
- `window.toolbar` — `NSToolbar` + `NSToolbarItem` + `NSToolbarItemGroup`.
- `window.contentViewController` — root VC.
- `window.contentLayoutGuide`/`contentLayoutRect` — content area guides.
- `window.isMovable`, `isMovableByWindowBackground`, `isExcludedFromWindowsMenu`, `isReleasedWhenClosed`.
- `window.level` — `.normal`, `.floating`, `.modalPanel`, `.statusBar`, `.popUpMenu`, `.screenSaver`, `.dock`, `.mainMenu`.
- `window.collectionBehavior` — `.fullScreenPrimary`, `.fullScreenAuxiliary`, `.canJoinAllSpaces`, `.moveToActiveSpace`, `.stationary`, `.ignoresCycle`, `.managed`, `.transient`, `.participatesInCycle`.
- `window.sharingType` — `.readOnly`, `.none` — screen sharing privacy.
- `window.animationBehavior` — `.default`, `.none`, `.documentWindow`, `.utilityWindow`.
- `window.isExcludedFromWindowsMenu` — hide from Window menu.
- `window.tabbingMode` — `.automatic`, `.preferred`, `.disallowed`.
- `window.titlebarAccessoryViewControllers` — titlebar accessory VCs.
- `window.addTitlebarAccessoryViewController` — add accessory to titlebar.
- `window.standardWindowButton` — access standard buttons for custom styling.
- `window.backgroundColor`, `isOpaque`, `hasShadow` — window appearance.
- `window.appearance` — NSAppearance for window.
- `window.effectiveAppearance` — resolved appearance.

## 6. Sheets & panels

```swift
// Sheet
let sheet = NSWindow(contentRect: NSRect(x: 0, y: 0, width: 400, height: 300),
  styleMask: [.titled, .closable], backing: .buffered, defer: false)
sheet.contentViewController = sheetVC
parentWindow.beginSheet(sheet) { response in ... }
// or
parentWindow.beginSheet(sheet, completionHandler: { response in ... })
// End:
parentWindow.endSheet(sheet, returnCode: .OK)
```

- `NSWindow.beginSheet(_:completionHandler:)` — attach sheet to window.
- `NSWindow.endSheet(_:returnCode:)` — dismiss sheet.
- `NSApplication.beginModalSession(for:)` — app-modal sheet.
- `NSApplication.runModal(for:)` — modal window.
- `NSApplication.stopModal(withCode:)` — stop modal.
- `NSWindowController.showWindow(_:)` — show window.
- `NSWindowController.close()` — close window.
- `NSWindowController.windowWillClose(_:)` — delegate.
- `NSPanel` — `NSPanel` subclass for utility panels: `nonactivatingPanel`, `utilityWindow`, `hidesOnDeactivate`, `becomesKeyOnlyIfNeeded`.
- `NSSavePanel`/`NSOpenPanel` — file panels: `canChooseFiles`, `canChooseDirectories`, `allowsMultipleSelection`, `allowedFileTypes`/`allowedContentTypes`, `directoryURL`, `nameFieldStringValue`, `runModal()`, `beginSheetModal(for:completionHandler:)`.
- `NSPrintPanel` — print settings.
- `NSColorPanel` — color picker.
- `NSFontPanel` — font picker.
- `NSAlert` — alert dialog: `alertStyle` (`.warning`, `.informational`, `.critical`), `messageText`, `informativeText`, `addButton(withTitle:)`, `accessoryView`, `runModal()`, `beginSheetModal(for:completionHandler:)`.

## 7. Drag & drop deep

```swift
// Source
let item = NSDraggingItem(pasteboardWriter: myObject)   // or NSPasteboardItem
item.draggingFrame = view.bounds
item.imageComponentsProvider = {
  let component = NSDraggingImageComponent(key: .icon)
  component.contents = image
  component.frame = CGRect(x: 0, y: 0, width: 32, height: 32)
  return [component]
}
view.beginDraggingSession(with: [item], event: event, source: self)

// Destination
view.registerForDraggedTypes([.string, .fileURL, NSPasteboard.PasteboardType("com.myapp.custom")])
// Implement NSDraggingDestination:
func draggingEntered(_ sender: NSDraggingInfo) -> NSDragOperation {
  guard sender.draggingPasteboard.canReadObject(forClasses: [NSURL.self], options: nil) else { return [] }
  return .copy
}
func draggingUpdated(_ sender: NSDraggingInfo) -> NSDragOperation { .copy }
func performDragOperation(_ sender: NSDraggingInfo) -> Bool {
  let pb = sender.draggingPasteboard
  if let urls = pb.readObjects(forClasses: [NSURL.self], options: nil) as? [URL] {
    // Handle dropped files
  }
  return true
}
```

- `NSDraggingSource`/`NSDraggingDestination`/`NSDraggingInfo` protocols.
- `NSDraggingItem` — `pasteboardWriter`, `draggingFrame`, `imageComponentsProvider`, `springLoadingHighlight`.
- `NSDraggingImageComponent` — drag image parts: `key` (`.icon`, `.label`, `.background`), `contents`, `frame`, `alignmentPoint`.
- `NSPasteboard.PasteboardType` — `.string`, `.URL`, `.fileURL`, `.tiff`, `.png`, `.color`, `.font`, `.html`, `.rtf`, `.pdf`, custom.
- `NSPasteboardItem` — multiple representations of one item.
- `NSPasteboard` — `general`, `find`, `ruler`, `font` named pasteboards; `clearContents`, `setString`, `setData`, `writeObjects`, `readObjects`, `propertyList(forType:)`, `setPropertyList`, `changeCount`.
- `NSFilePromiseProvider` — promised files (async file provision).
- `NSPasteboardItemDataProvider` — custom data provider.
- Spring loading: `springLoadingHighlight` on `NSDraggingItem`.

## 8. Pasteboard deep

```swift
let pb = NSPasteboard.general
pb.clearContents()
pb.setString("text", forType: .string)
pb.setData(data, forType: .fileURL)
pb.setPropertyList(dict, forType: .fileURL)
pb.writeObjects([url, image, string])
pb.readObjects(forClasses: [NSURL.self, NSImage.self, NSString.self], options: nil)
pb.changeCount  // increments on every write — poll for changes
pb.name  // NSPasteboard.Name.general/.find/.ruler/.font
```

- `NSPasteboard.Name` — `.general`, `.find`, `.font`, `.ruler`, `.drag`.
- `NSPasteboardItem` — `writeObjects`, `setString`, `setData`, `propertyList(forType:)`, `setPropertyList`.
- `NSPasteboardItemDataProvider` — `pasteboard(_:item:provideDataForType:)` for lazy data.
- `NSFilePromiseProvider` — `filePromiseProvider(_:fileNameForType:)` — promised file names.
- Find pasteboard: `NSTextView.usesFindBar` + `.find` pasteboard for find/replace.
- Change count: `pb.changeCount` — poll for pasteboard changes (e.g. clipboard managers).
- `NSPasteboard` `availableType(from:)` — check available types.

## 9. Services & login items

### Services
- `NSServices` in Info.plist — expose app functionality to Services menu.
- `NSRegisterServices` — register services at runtime.
- `NSPasteboard` `serviceProvider` — provide data to services.

### Login items
```swift
import ServiceManagement

// Modern (macOS 13+)
let status = SMAppService.mainApp.status  // .enabled, .requiresApproval, .notFound
try SMAppService.mainApp.register()
try SMAppService.mainApp.unregister()

// Login item (helper app)
let loginItem = SMAppService.loginItem(identifier: "com.example.helper")
try loginItem.register()

// Daemon/agent (launchd)
let daemon = SMAppService.daemon(plistName: "com.example.daemon.plist")
try daemon.register()

// Agent (user agent)
let agent = SMAppService.agent(plistName: "com.example.agent.plist")
try agent.register()
```

- `SMAppService` — `mainApp`, `loginItem(identifier:)`, `daemon(plistName:)`, `agent(plistName:)`; `.status` (`.enabled`, `.requiresApproval`, `.notFound`).
- `SMAppService.Status` — `.enabled`, `.requiresApproval`, `.notFound`.
- Legacy: `LSSharedFileList` — deprecated; use `SMAppService`.

### XPC services
- `NSXPCConnection`/`NSXPCInterface`/`NSXPCListener` — inter-process communication.
- `NSXPCListener` — `listener(withMachServiceName:)` or `anonymous`; `delegate` for new connections.
- `NSXPCInterface` — `interfaceWithProtocol:` for typed interface.
- `NSXPCConnection` — `init(machServiceName:)` or `init(listenerEndpoint:)`; `remoteObjectProxy`, `exportedObject`, `resume`, `invalidate`.
- `NSXPCListenerEndpoint` — endpoint for connection.

## 10. Printing deep

```swift
let printInfo = NSPrintInfo.shared
printInfo.jobDisposition = .save
printInfo.dictionary()[NSPrintInfo.AttributeKey.jobSavingFileNameURL] = fileURL
printInfo.dictionary()[NSPrintInfo.AttributeKey.jobSavingURL] = fileURL

let operation = NSPrintOperation(view: viewToPrint, printInfo: printInfo)
operation.showsPrintPanel = true
operation.showsProgressPanel = true
operation.runModal()
// or
operation.beginSheetModal(for: window) { response, success, error in ... }
```

- `NSPrintOperation` — `view`/`viewController`/`printInfo`/`printPanel`/`printService`; `runModal()`, `beginSheetModal(for:completionHandler:)`, `runOperationModal`.
- `NSPrintInfo` — `jobDisposition` (`.save`, `.preview`, `.print`, `.spill`, `.delete`), `dictionary()` for attributes, `horizontalPagination`, `verticalPagination`, `orientation`, `paperSize`, `topMargin`, `bottomMargin`, `leftMargin`, `rightMargin`, `isHorizontallyCentered`, `isVerticallyCentered`, `scalingFactor`.
- `NSPrintPanel` — `options` (`.showsCopies`, `.showsPageRange`, `.showsPaperSize`, `.showsOrientation`, `.showsScaling`), `runModal()`, `beginSheetModal(for:completionHandler:)`.
- `NSPrintService` — `availablePrintServices`, `printerWithName`, `isAvailable`.
- `NSPrintOperation` `PDF` attribute — generate PDF: `printInfo.dictionary()[NSPrintInfo.AttributeKey.jobSavingURL] = pdfURL` + `jobDisposition = .save`.

## 11. Undo/redo

- `NSUndoManager` — `registerUndo(withTarget:handler:)`, `registerUndo(withTarget:selector:object:)`, `undo()`, `redo()`, `canUndo`, `canRedo`, `groupsByEvent`, `levelsOfUndo`, `removeAllActions`.
- `NSResponder` `undoManager` — responder chain undo manager.
- `NSDocument` `undoManager` — document undo manager.
- `NSUndoManager` `beginUndoGrouping()`/`endUndoGrouping()` — group undo actions.
- `NSUndoManager` `setActionName(_:)` — action name for menu.
- `NSUndoManager` `undoActionName`, `redoActionName` — localized action names.
- `NSUndoManager` `isUndoing`, `isRedoing` — state flags.
- `NSUndoManager` `runLoopModes` — run loop modes for undo.
- `NSUndoManager` `automaticallyGroupsEvents` — auto-group by event.
- `NSUndoManager` `groupsByEvent` — group by event.
- `NSUndoManager` `levelsOfUndo` — max undo levels.
- `NSUndoManager` `removeAllActions` — clear undo stack.
- `NSUndoManager` `undoNestedGroup()`/`redoNestedGroup()` — nested groups.
- `NSUndoManager` `prepare(withInvocation:)` — prepare undo action.

## 12. Document architecture deep

- `NSDocument` — `fileURL`, `fileName`, `displayName`, `isDocumentEdited`, `save(to:ofType:for:)`, `read(from:ofType:)`, `autosavesInPlace`, `keepsUndoManager`, `hasUndoManager`, `updateChangeCount`, `undoManager`, `windowControllers`, `makeWindowControllers()`, `revertToSaved()`, `saveToURL(_:ofType:for:)`, `writeSafely(to:ofType:for:)`, `readSafely(from:ofType:)`, `canWriteToURL(_:ofType:for:)`, `canReadFromURL(_:ofType:)`, `shouldAutosave`? — check.
- `NSDocumentController` — `openDocument(withContentsOf:display:completionHandler:)`, `newDocument`, `documentFor(window:)`, `documents` array, `currentDocument`, `documentShouldClose`, `documentWillClose`, `documentDidClose`.
- `NSPersistentDocument` — `autosavesInPlace`, `autosavingFileType`, `isDocumentEdited`.
- `NSFileCoordinator` — `coordinate(readingItemAt:options:error:byAccessor:)`, `coordinate(writingItemAt:options:error:byAccessor:)`, `prepareForReading`, `prepareForWriting`.
- `NSFilePresenter` — `presentedItemURL`, `presentedItemOperationQueue`, `presentedItemDidChange`, `presentedItemDidMoveToURL`, `presentedItemDidGainFocus`, `presentedItemDidLoseFocus`, `savePresentedItemChanges`, `relinquishPresentedItemToURL`, `presentedItemDidReceiveUnshare`? — check.
- `NSDocument` `autosavesInPlace` — autosave in place vs versions.
- `NSDocument` `keepsUndoManager` — keep undo manager across saves.
- `NSDocument` `hasUndoManager` — has undo manager.
- `NSDocument` `updateChangeCount(_:)` — update change count.
- `NSDocument` `isDocumentEdited` — document edited flag.
- `NSDocument` `fileModificationDate` — file modification date.
- `NSDocument` `fileType` — document type.
- `NSDocument` `autosavingFileType` — autosave file type.
- `NSDocument` `autosavesInPlace` — autosave in place.
- `NSDocument` `windowControllers` — window controllers array.
- `NSDocument` `makeWindowControllers()` — create window controllers.
- `NSDocument` `revertToSaved()` — revert to saved.
- `NSDocument` `saveToURL(_:ofType:for:)` — save to URL.
- `NSDocument` `writeSafely(to:ofType:for:)` — safe write.
- `NSDocument` `readSafely(from:ofType:)` — safe read.
- `NSDocument` `canWriteToURL(_:ofType:for:)` — can write.
- `NSDocument` `canReadFromURL(_:ofType:)` — can read.
- `NSDocument` `shouldAutosave`? — check.
- `NSDocument` `updateChangeCount` — update change count.
- `NSDocument` `isDocumentEdited` — document edited flag.
- `NSDocument` `fileModificationDate` — file modification date.
- `NSDocument` `fileType` — document type.
- `NSDocument` `autosavingFileType` — autosave file type.
- `NSDocument` `autosavesInPlace` — autosave in place.
- `NSDocument` `windowControllers` — window controllers array.
- `NSDocument` `makeWindowControllers()` — create window controllers.
- `NSDocument` `revertToSaved()` — revert to saved.
- `NSDocument` `saveToURL(_:ofType:for:)` — save to URL.
- `NSDocument` `writeSafely(to:ofType:for:)` — safe write.
- `NSDocument` `readSafely(from:ofType:)` — safe read.
- `NSDocument` `canWriteToURL(_:ofType:for:)` — can write.
- `NSDocument` `canReadFromURL(_:ofType:)` — can read.
- `NSDocument` `shouldAutosave`? — check.

## 13. State restoration

- `NSWindow` `isRestorable`, `restorationClass`, `restorableState`.
- `NSWindowController` `restorationClass`.
- `NSView` `restorableState` — `encodeRestorableState(with:)`/`decodeRestorableState(with:)`.
- `NSViewController` `restorableState` — same.
- `NSWindow` `setFrameAutosaveName(_:)` — remember frame.
- `NSWindow` `setFrameUsingName(_:)` — restore frame.
- `NSWindow` `frameAutosaveName` — autosave name.
- `NSWindow` `isRestorable` — restore state.
- `NSWindow` `restorationClass` — restoration class.
- `NSWindow` `restorableState` — restorable state.
- `NSWindow` `restorationClass` — restoration class.
- `NSWindow` `restorableState` — restorable state.
- `NSWindow` `setFrameAutosaveName(_:)` — remember frame.
- `NSWindow` `setFrameUsingName(_:)` — restore frame.
- `NSWindow` `frameAutosaveName` — autosave name.
- `NSWindow` `isRestorable` — restore state.
- `NSWindow` `restorationClass` — restoration class.
- `NSWindow` `restorableState` — restorable state.
- `NSWindow` `restorationClass` — restoration class.
- `NSWindow` `restorableState` — restorable state.

## 14. Accessibility (NSAccessibility)

- `NSAccessibility` protocol — `accessibilityRole()`, `accessibilityLabel()`, `accessibilityValue()`, `accessibilityHelp()`, `accessibilityChildren()`, `accessibilityParent()`, `accessibilityFrame()`, `accessibilityFocusedUIElement()`, `accessibilityFocusedUIElement()`, `accessibilityPerformPress()`, `accessibilityPerformIncrement()`, `accessibilityPerformDecrement()`, `accessibilityPerformShowMenu()`, `accessibilityPerformPick()`, `accessibilityIsIgnored()`, `accessibilityElementCount()`, `accessibilityChildrenInNavigationOrder()`, `accessibilityIndexOfChild()`, `accessibilityHitTest(_:)`, `accessibilityArrayAttributeValues(_:forParameter:)`, `accessibilityParameterizedAttributeNames(for:)`, `accessibilityParameterizedAttributeValues(for:for:)`.
- `NSAccessibilityElement` — custom accessibility elements: `accessibilityRole`, `accessibilityLabel`, `accessibilityValue`, `accessibilityHelp`, `accessibilityFrame`, `accessibilityParent`, `accessibilityChildren`.
- `NSAccessibility` `post(notification:argument:)` — post notifications: `.layoutChanged`, `.screenChanged`, `.announcement`, `.created`, `.selectedChildrenChanged`, `.titleChanged`, `.valueChanged`, `.selectedChildrenChanged`, `.menuClosed`, `.menuOpened`, `.rowCountChanged`, `.rowCollapsed`, `.rowExpanded`, `.selectedCellsChanged`, `.selectedChildrenChanged`, `.selectedRowsChanged`, `.selectedTextChanged`, `.selectedTextMarkerRangeChanged`, `.sortedChildrenChanged`, `.valueChanged`, `.windowCreated`, `.windowMoved`, `.windowResized`, `.windowMiniaturized`, `.windowDeminiaturized`, `.windowClosed`, `.windowFocused`, `.windowUnfocused`, `.mainWindowChanged`, `.focusedUIElementChanged`, `.focusedWindowChanged`, `.applicationActivated`, `.applicationDeactivated`, `.sheetCreated`, `.sheetMoved`, `.sheetResized`, `.sheetClosed`, `.rowExpanded`, `.rowCollapsed`, `.rowCountChanged`, `.selectedCellsChanged`, `.selectedChildrenChanged`, `.selectedRowsChanged`, `.selectedTextChanged`, `.selectedTextMarkerRangeChanged`, `.sortedChildrenChanged`, `.valueChanged`, `.windowCreated`, `.windowMoved`, `.windowResized`, `.windowMiniaturized`, `.windowDeminiaturized`, `.windowClosed`, `.windowFocused`, `.windowUnfocused`, `.mainWindowChanged`, `.focusedUIElementChanged`, `.focusedWindowChanged`, `.applicationActivated`, `.applicationDeactivated`, `.sheetCreated`, `.sheetMoved`, `.sheetResized`, `.sheetClosed`.
- `NSAccessibility` `isVoiceOverEnabled`? — `NSWorkspace.shared.isVoiceOverEnabled`? — check.
- `NSAccessibility` `isDifferentiateWithoutColorEnabled`? — `NSWorkspace.shared.accessibilityDisplayShouldDifferentiateWithoutColor`? — check.
- `NSAccessibility` `isReduceMotionEnabled`? — `NSWorkspace.shared.accessibilityDisplayShouldReduceMotion`? — check.
- `NSAccessibility` `isReduceTransparencyEnabled`? — `NSWorkspace.shared.accessibilityDisplayShouldReduceTransparency`? — check.
- `NSAccessibility` `isInvertColorsEnabled`? — `NSWorkspace.shared.accessibilityDisplayShouldInvertColors`? — check.
- `NSAccessibility` `isGrayscaleEnabled`? — `NSWorkspace.shared.accessibilityDisplayShouldGrayscale`? — check.
- `NSAccessibility` `isClosedCaptioningEnabled`? — `NSWorkspace.shared.accessibilityDisplayShouldClosedCaptioning`? — check.
- `NSAccessibility` `isMonoAudioEnabled`? — `NSWorkspace.shared.accessibilityDisplayShouldMonoAudio`? — check.
- `NSAccessibility` `isVideoAutoplayEnabled`? — `NSWorkspace.shared.accessibilityDisplayShouldVideoAutoplay`? — check.
- `NSAccessibility` `isAudioDescriptionEnabled`? — `NSWorkspace.shared.accessibilityDisplayShouldAudioDescription`? — check.
- `NSAccessibility` `isSpatialAudioEnabled`? — `NSWorkspace.shared.accessibilityDisplayShouldSpatialAudio`? — check.

## 15. Catalyst deep

- **Mac Catalyst**: iPad/iOS apps running on macOS with Mac-idiomatic UI.
- `TARGET_OS_MACCATALYST` conditional compilation.
- `#if targetEnvironment(macCatalyst)` for Catalyst-specific code.
- `UIDevice.current.userInterfaceIdiom` — `.mac` for Catalyst.
- `UIKeyboardHIDUsage` — hardware keyboard support.
- `UIMenuBuilder`/`UIMenuSystem` — menu building on Mac.
- `UIKeyCommand` — keyboard shortcuts.
- `UIPointerInteraction`/`UIPointerStyle` — pointer support.
- `UIHoverGestureRecognizer` — hover effects.
- `UIViewController` `preferredScreenEdgesDeferringSystemGestures` — edge gesture deferral.
- `UIViewController` `modalPresentationStyle` — Mac-idiomatic presentation.
- `UIViewController` `modalTransitionStyle` — Mac-idiomatic transitions.
- `UIViewController` `modalPresentationCapturesStatusBarAppearance` — status bar during modal.
- `UIViewController` `definesPresentationContext` — presentation context.
- `UIViewController` `providesPresentationContextTransitionStyle` — transition style.
- `UIViewController` `modalInPresentation` — prevent swipe-to-dismiss.
- `UIViewController` `isModalInPresentation` — same.
- `UIViewController` `preferredStatusBarStyle`/`prefersHomeIndicatorAutoHidden`/`prefersStatusBarHidden` — status bar control.
- `UIViewController` `supportedInterfaceOrientations` — orientation support.
- `UIViewController` `preferredInterfaceOrientationForPresentation` — orientation.
- `UIViewController` `shouldAutorotate` — autorotation.
- `UIViewController` `viewWillTransition(to:with:)` — transition callback.
- `UIViewController` `traitCollectionDidChange(_:)` — trait change callback.
- `UIViewController` `willTransition(to:with:)` — transition coordinator.
- `UIViewController` `viewWillLayoutSubviews`/`viewDidLayoutSubviews` — layout callbacks.
- `UIViewController` `updateViewConstraints` — constraint update.
- `UIViewController` `loadView` — view loading.
- `UIViewController` `viewDidLoad` — view loaded.
- `UIViewController` `viewWillAppear`/`viewDidAppear`/`viewWillDisappear`/`viewDidDisappear` — appearance callbacks.
- `UIViewController` `didReceiveMemoryWarning` — memory warning.
- `UIViewController` `isMovingToParent`/`isMovingFromParent` — containment.
- `UIViewController` `willMove(toParent:)`/`didMove(toParent:)` — containment callbacks.
- `UIViewController` `addChild`/`removeFromParent` — containment.
- `UIViewController` `present`/`dismiss` — presentation.
- `UIViewController` `navigationController` — navigation.
- `UIViewController` `tabBarController` — tab bar.
- `UIViewController` `splitViewController` — split view.
- `UIViewController` `presentedViewController`/`presentingViewController` — presentation.
- `UIViewController` `children` — child VCs.
- `UIViewController` `parent` — parent VC.
- `UIViewController` `definesContext` — context definition.
- `UIViewController` `transitionCoordinator` — transition coordinator.
- `UIViewController` `preferredContentSize` — preferred content size.
- `UIViewController` `contentSize` — content size.
- `UIViewController` `modalPresentationStyle` — presentation style.
- `UIViewController` `modalTransitionStyle` — transition style.
- `UIViewController` `modalPresentationCapturesStatusBarAppearance` — status bar.
- `UIViewController` `definesPresentationContext` — context.
- `UIViewController` `providesPresentationContextTransitionStyle` — transition style.
- `UIViewController` `modalInPresentation` — modal.
- `UIViewController` `isModalInPresentation` — modal.
- `UIViewController` `preferredStatusBarStyle` — status bar.
- `UIViewController` `prefersHomeIndicatorAutoHidden` — home indicator.
- `UIViewController` `prefersStatusBarHidden` — status bar.
- `UIViewController` `supportedInterfaceOrientations` — orientations.
- `UIViewController` `preferredInterfaceOrientationForPresentation` — orientation.
- `UIViewController` `shouldAutorotate` — autorotate.
- `UIViewController` `viewWillTransition` — transition.
- `UIViewController` `traitCollectionDidChange` — traits.
- `UIViewController` `willTransition` — coordinator.
- `UIViewController` `viewWillLayoutSubviews`/`viewDidLayoutSubviews` — layout.
- `UIViewController` `updateViewConstraints` — constraints.
- `UIViewController` `loadView` — load.
- `UIViewController` `viewDidLoad` — loaded.
- `UIViewController` `viewWillAppear`/`viewDidAppear`/`viewWillDisappear`/`viewDidDisappear` — appearance.
- `UIViewController` `didReceiveMemoryWarning` — memory.
- `UIViewController` `isMovingToParent`/`isMovingFromParent` — containment.
- `UIViewController` `willMove`/`didMove` — containment.
- `UIViewController` `addChild`/`removeFromParent` — containment.
- `UIViewController` `present`/`dismiss` — presentation.
- `UIViewController` `navigationController` — navigation.
- `UIViewController` `tabBarController` — tab.
- `UIViewController` `splitViewController` — split.
- `UIViewController` `presentedViewController`/`presentingViewController` — presentation.
- `UIViewController` `children` — children.
- `UIViewController` `parent` — parent.
- `UIViewController` `definesContext` — context.
- `UIViewController` `transitionCoordinator` — coordinator.
- `UIViewController` `preferredContentSize` — size.
- `UIViewController` `contentSize` — size.

## 16. Common advanced AppKit bugs → fixes

| Bug | Cause | Fix |
|---|---|---|
| Menu item not validating | `validateMenuItem` not implemented or returning wrong value | implement validation, check responder chain |
| Window not key | `makeKeyAndOrderFront` not called, `canBecomeKey` false | call `makeKeyAndOrderFront`, check `canBecomeKey` |
| Sheet not dismissing | `endSheet` not called, wrong window | call `endSheet`, check parent window |
| Text view scrolling wrong | `NSTextView` not in `NSScrollView` | embed in scroll view, set `documentView` |
| Bindings not updating | property not `@objc dynamic`, KVO not firing | mark `@objc dynamic`, check KVO |
| Drag image wrong | `imageComponentsProvider` not set | set provider for custom drag image |
| Pasteboard not updating | `clearContents` not called, wrong type | call `clearContents`, check `PasteboardType` |
| Print output wrong | wrong `NSPrintInfo` settings | check `NSPrintInfo` attributes |
| Undo not working | `undoManager` not registered, action not registered | register undo actions, check `canUndo` |
| Document not saving | `save(to:ofType:for:)` not implemented | implement save/read methods |
| State not restoring | `restorableState` not implemented | implement `encodeRestorableState`/`decodeRestorableState` |
| Accessibility broken | missing `NSAccessibility` methods | implement `NSAccessibility` protocol methods |
| Catalyst app crashes | unsupported iOS API | check Catalyst limitations, use `#if targetEnvironment(macCatalyst)` |
| Memory leak | retain cycle via delegate/block | use `weak` delegates, invalidate timers |
| Performance slow | main thread blocking, excessive layout | move work to background, optimize layout |
