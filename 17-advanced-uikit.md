# Advanced UIKit — Internals & Expert APIs

Deep-dive beyond `04-uikit.md`: TextKit, custom transitions, compositional layout, diffable data source, Auto Layout engine, responder chain, gestures, scroll physics, appearance, keyboard, drag/drop, modern cell APIs, performance.

## 1. Text system — TextKit 2

UIKit text uses **TextKit 2** by default (iOS 15+): `NSTextContentStorage` + `NSTextLayoutManager` + `NSTextContainer` + `NSTextLayoutFragment`.

```swift
let textView = UITextView()
textView.isScrollEnabled = false
// TextKit 2 stack (iOS 15+):
let storage = NSTextStorage()
let layoutManager = NSTextLayoutManager()
let container = NSTextContainer(size: CGSize(width: 300, height: .greatestFiniteMagnitude))
container.lineFragmentPadding = 0
layoutManager.textContainer = container
storage.addLayoutManager(layoutManager)
// Add text view's own textContainer? — UITextView manages its own; for custom:
```

- **TextKit 1 (legacy)**: `NSTextStorage` → `NSLayoutManager` → `NSTextContainer` → `NSTextLayoutManager`? — TextKit 1: `NSTextStorage` + `NSLayoutManager` + `NSTextContainer`; TextKit 2 replaces NSLayoutManager with NSTextLayoutManager + NSTextContentManager.
- `UITextView` uses TextKit 2 when `textView.textLayoutManager != nil` (iOS 16+ default for new text views).
- `NSTextAttachment` — inline images/objects: `NSTextAttachment(image:)`, `bounds`, `fileWrapper`.
- `NSTextList` — ordered/unordered lists in attributed strings.
- `NSParagraphStyle` — `NSMutableParagraphStyle`: `alignment`, `lineSpacing`, `paragraphSpacing`, `lineBreakMode`, `hyphenationFactor`, `tabStops`, `defaultTabInterval`, `allowsDefaultTighteningForTruncation`, `usesDefaultHyphenation`, `lineHeightMultiple`.
- `NSAttributedString` attributes: `.font`, `.foregroundColor`, `.backgroundColor`, `.kern`, `.ligature`, `.baselineOffset`, `.underlineStyle`, `.strikethroughStyle`, `.attachment`, `.link`, `.shadow`, `.strokeColor`, `.strokeWidth`, `.obliqueness`, `.expansion`, `.paragraphStyle`, `.inlinePresentationIntent` (iOS 15+), `.textEffect`? — check.
- `NSMutableAttributedString` — efficient editing: `replaceCharacters`, `setAttributes`, `addAttribute`, `removeAttribute`, `enumerateAttributes`.
- `NSTextCheckingResult`/`NSDataDetector` — links, phone numbers, dates detection: `NSDataDetector(types:)` → `matches(in:options:range:)`.
- Exclusion paths: `textContainer.exclusionPaths` — wrap text around shapes (TextKit 1); TextKit 2 uses `NSTextLayoutFragment` exclusions? — `textLayoutManager.textContainer.exclusionPaths` still works.
- `UITextInput` protocol — marked text (IME composition), `beginningOfDocument`, `endOfDocument`, `position(from:offset:)`, `text(in:)`, `replace(_:withText:)`, `setMarkedText`, `unmarkText`, `forwardDelete`.
- `UITextInputTraits` — `keyboardType`, `returnKeyType`, `enablesReturnKeyAutomatically`, `isSecureTextEntry`, `autocapitalizationType`, `autocorrectionType`, `spellCheckingType`, `smartQuotesType`, `smartDashesType`, `smartInsertDeleteType`, `textContentType`, `passwordRules`.
- `UITextInputDelegate` — `textWillChange`, `textDidChange`, `selectionWillChange`, `selectionDidChange`.

## 2. Custom view controller transitions

```swift
class SlideAnimator: NSObject, UIViewControllerAnimatedTransitioning {
  let isPresenting: Bool
  init(isPresenting: Bool) { self.isPresenting = isPresenting }

  func transitionDuration(using context: UIViewControllerContextTransitioning?) -> TimeInterval { 0.3 }

  func animateTransition(using context: UIViewControllerContextTransitioning) {
    let container = context.containerView
    let toView = context.view(forKey: .to)!
    let fromView = context.view(forKey: .from)!

    if isPresenting {
      container.addSubview(toView)
      toView.frame = container.bounds.offsetBy(dx: container.bounds.width, dy: 0)
      UIView.animate(withDuration: 0.3, delay: 0, options: .curveEaseOut) {
        toView.frame = container.bounds
      } completion: { _ in
        context.completeTransition(!context.transitionWasCancelled)
      }
    } else {
      UIView.animate(withDuration: 0.3, delay: 0, options: .curveEaseIn) {
        fromView.frame = container.bounds.offsetBy(dx: container.bounds.width, dy: 0)
      } completion: { _ in
        context.completeTransition(!context.transitionWasCancelled)
      }
    }
  }
}

class SlideInteraction: NSObject, UIViewControllerInteractiveTransitioning {
  var animator: UIViewControllerAnimatedTransitioning?
  var interactionController: UIPercentDrivenInteractiveTransition?
  var shouldComplete = false

  func startInteractiveTransition(_ transitionContext: UIViewControllerContextTransitioning) {
    interactionController = UIPercentDrivenInteractiveTransition()
    interactionController?.completionCurve = .easeOut
    animator?.animateTransition(using: transitionContext)
  }
  // Update percent: interactionController?.update(translation / containerWidth)
  // Finish/cancel: interactionController?.finish() / .cancel()
}

// Delegate
class TransitionDelegate: NSObject, UIViewControllerTransitioningDelegate {
  func animationController(forPresented presented: UIViewController,
    presenting: UIViewController, source: UIViewController) -> UIViewControllerAnimatedTransitioning? {
    SlideAnimator(isPresenting: true)
  }
  func animationController(forDismissed dismissed: UIViewController) -> UIViewControllerAnimatedTransitioning? {
    SlideAnimator(isPresenting: false)
  }
  func interactionControllerForPresentation(using animator: UIViewControllerAnimatedTransitioning) -> UIViewControllerInteractiveTransitioning? {
    slideInteraction
  }
}

// Usage
vc.modalPresentationStyle = .custom
vc.transitioningDelegate = transitionDelegate
vc.modalPresentationCapturesStatusBarAppearance = true
present(vc, animated: true)
```

- `UIViewControllerAnimatedTransitioning` — `transitionDuration`, `animateTransition`.
- `UIViewControllerInteractiveTransitioning` — `startInteractiveTransition`.
- `UIPercentDrivenInteractiveTransition` — drives interactive transition by percent (0-1).
- `UIViewControllerTransitioningDelegate` — provide animators for present/dismiss.
- `UIPresentationController` — custom presentation: frame, chrome, dismissal.
- `UIViewPropertyAnimator` — interruptible/reversible animations: `UIViewPropertyAnimator(duration:timingParameters:)`, `startAnimation`, `pauseAnimation`, `stopAnimation`, `fractionComplete`, `isReversed`.
- `UIView.animateKeyframes` — keyframe animations: `withDuration`, `delayOptions`, `animations` with `UIView.addKeyframe(withRelativeStartTime:relativeDuration:animations:)`.
- `UIView.animate(withDuration:delay:usingSpringWithDamping:initialSpringVelocity:options:animations:completion:)` — spring params.

## 3. UICollectionView Compositional Layout deep

```swift
let itemSize = NSCollectionLayoutSize(widthDimension: .fractionalWidth(1.0),
  heightDimension: .fractionalHeight(1.0))
let item = NSCollectionLayoutItem(layoutSize: itemSize)
item.contentInsets = NSDirectionalEdgeInsets(top: 4, leading: 4, bottom: 4, trailing: 4)

let groupSize = NSCollectionLayoutSize(widthDimension: .fractionalWidth(1.0),
  heightDimension: .absolute(200))
let group = NSCollectionLayoutGroup.horizontal(layoutSize: groupSize, subitems: [item, item, item])

let section = NSCollectionLayoutSection(group: group)
section.contentInsets = NSDirectionalEdgeInsets(top: 16, leading: 16, bottom: 16, trailing: 16)
section.interGroupSpacing = 12
section.orthogonalScrollingBehavior = .continuous  // horizontal scroll within vertical
section.visibleItemsInvalidationHandler = { items, offset, environment in
  // Called per frame during scroll — for parallax effects
  for item in items {
    item.transform = CGAffineTransform(translationX: 0, y: offset.y * 0.5)
  }
}

// Boundary supplementary items (headers/footers)
let headerSize = NSCollectionLayoutSize(widthDimension: .fractionalWidth(1.0), heightDimension: .absolute(44))
let header = NSCollectionLayoutBoundarySupplementaryItem(layoutSize: headerSize,
  elementKind: UICollectionView.elementKindSectionHeader,
  alignment: .top)
section.boundarySupplementaryItems = [header]

// Decoration views (backgrounds)
let backgroundSize = NSCollectionLayoutSize(widthDimension: .fractionalWidth(1.0),
  heightDimension: .fractionalHeight(1.0))
let background = NSCollectionLayoutDecorationItem(backgroundColor: .systemBackground)
section.decorationItems = [background]

let layout = UICollectionViewCompositionalLayout(section: section)
collectionView.collectionViewLayout = layout
collectionView.register(UICollectionReusableView.self,
  forDecorationViewOfKind: "background")
```

- `NSCollectionLayoutSize` — `.absolute`, `.fractionalWidth`, `.fractionalHeight`, `.estimated`.
- `NSCollectionLayoutItem` — `contentInsets`, `edgeSpacing`, `contentInsetsReference`.
- `NSCollectionLayoutGroup` — `.horizontal`, `.vertical`, `.custom` (layout closure); `interItemSpacing` (`.fixed`, `.flexible`).
- `NSCollectionLayoutSection` — `contentInsets`, `interGroupSpacing`, `orthogonalScrollingBehavior` (`.none`, `.continuous`, `.continuousGroupLeading`, `.paging`, `.groupPaging`), `visibleItemsInvalidationHandler`, `boundarySupplementaryItems`, `decorationItems`, `contentInsetsReference`.
- `NSCollectionLayoutBoundarySupplementaryItem` — headers/footers: `alignment` (`.top`, `.bottom`, `.leading`, `.trailing`, `.topLeading` etc.), `pinToVisibleBounds`, `absoluteOffset`.
- `NSCollectionLayoutDecorationItem` — background/foreground decorations.
- `NSCollectionLayoutAnchor` — custom supplementary item positioning.
- Modern cell registration: `UICollectionView.CellRegistration` — replaces `dequeueReusableCell`:
  ```swift
  let registration = UICollectionView.CellRegistration<MyCell, Item> { cell, indexPath, item in
    cell.configure(with: item)
  }
  dataSource = UICollectionViewDiffableDataSource<Section, Item>(collectionView: cv) { cv, ip, item in
    cv.dequeueConfiguredReusableCell(using: registration, for: ip, item: item)
  }
  ```
- Supplementary registration: `UICollectionView.SupplementaryRegistration`.
- `NSDiffableDataSourceSnapshot` — `appendSections`, `appendItems`, `deleteSections`, `deleteItems`, `moveSection`, `moveItem`, `reloadSections`, `reloadItems`, `reconfigureItems` (iOS 15+ — update without animation).

## 4. Auto Layout engine deep

### Constraint priorities
- `UILayoutPriority` 1-1000: `.required` (1000), `.defaultHigh` (750), `.defaultLow` (250), `.fittingSizeLevel` (50).
- Hugging (250 default) — resist stretching; Compression resistance (750 default) — resist shrinking.
- Higher priority wins when constraints conflict; lower-priority constraints break first.

### Layout passes
```
setNeedsLayout() → layout pass scheduled
  → updateConstraints() (constraint changes)
  → layoutSubviews() (frame changes)
layoutIfNeeded() → forces immediate layout pass
```

- `setNeedsUpdateConstraints()` + `updateConstraints()` — batch constraint changes.
- `layoutIfNeeded()` inside `UIView.animate` — animates layout changes.
- `systemLayoutSizeFitting(_:withHorizontalFittingPriority:verticalFittingPriority:)` — measure view size.
- `sizeThatFits(_:)` / `sizeThatFits(_:uiView:context:)` in SwiftUI representables.
- `fittingSize` — intrinsic fitting.
- `invalidateIntrinsicContentSize()` — mark intrinsic size stale.

### Performance
- Constraint count matters — flatten hierarchies, use `UIStackView` for common cases.
- Avoid constraint churn in scroll views; precompute where possible.
- `translatesAutoresizingMaskIntoConstraints = false` for all programmatic constraints.
- Ambiguous layouts log "Unable to simultaneously satisfy constraints" — fix priorities or remove conflicts.
- `UIViewAlertForUnsatisfiableConstraints` breakpoint catches runtime conflicts.

### Layout margins & guides
- `layoutMargins` — `directionalLayoutMargins` (NSDirectionalEdgeInsets); `preservesSuperviewLayoutMargins`.
- `layoutMarginsGuide` — anchor to margins.
- `readableContentGuide` — readable width guide.
- `safeAreaLayoutGuide` — safe area.
- `UILayoutGuide` — custom guides without dummy views: `NSLayoutGuide`/`UILayoutGuide`, `addLayoutGuide`, constraints to guide anchors.

## 5. Responder chain & touch handling deep

### Hit testing
```swift
override func hitTest(_ point: CGPoint, with event: UIEvent?) -> UIView? {
  guard isUserInteractionEnabled, !isHidden, alpha > 0.01 else { return nil }
  guard point(inside: point, with: event) else { return nil }
  for subview in subviews.reversed() {   // topmost first
    let converted = convert(point, to: subview)
    if let hit = subview.hitTest(converted, with: event) { return hit }
  }
  return self
}

override func point(inside point: CGPoint, with event: UIEvent?) -> Bool {
  // Custom hit area (e.g. circular button)
  let center = bounds.mid
  return hypot(point.x - center.x, point.y - center.y) < bounds.width / 2
}
```

- `hitTest` — called recursively; return deepest view containing point.
- `point(inside:)` — override for custom hit shapes.
- `isUserInteractionEnabled`, `isHidden`, `alpha` — checked by default hitTest.

### Touch delivery
- `touchesBegan/Moved/Ended/Cancelled` on UIResponder — raw touch events.
- `UITouch` — `location(in:)`, `previousLocation(in:)`, `timestamp`, `phase`, `tapCount`, `force`, `altitudeAngle`, `azimuthAngle(in:)` (Pencil), `type` (`.direct`, `.indirect`, `.pencil`, `.stylus`).
- `UIEvent` — `allTouches`, `coalescedTouches(for:)`, `predictedTouches(for:)` (Pencil prediction).
- `exclusiveTouch` — view gets all touches in its bounds.
- `delaysTouchesBegan`/`delaysTouchesEnded`/`cancelsTouchesInView` on gesture recognizers — touch delivery timing.
- `gestureRecognizerShouldBegin`, `shouldReceive touch`, `shouldRecognizeSimultaneously`, `shouldRequireFailureOf`, `shouldBeRequiredToFailBy` — delegate control.
- Gesture recognizer state machine: `.possible` → `.began` → `.changed` → `.ended`/`.cancelled`/`.failed`.

### Pencil
- `UIPencilInteraction` — delegate for pencil events: `pencilInteraction(_:didReceiveTap:)`, `pencilInteraction(_:didReceiveSqueeze:)`, `pencilInteraction(_:didReceiveScroll:)`.
- `UITouch.type == .pencil` — detect pencil touches.
- `altitudeAngle`, `azimuthAngle` — pencil tilt/rotation.
- `coalescedTouches`, `predictedTouches` — high-frequency input.
- `UIPencilInteraction.prefersPencilOnlyDrawing`? — `UIPencilInteraction` has `prefersPencilOnlyDrawing`? — check: it's on `PKCanvasView.drawingPolicy`.

## 6. Scroll physics deep

- `contentInsetAdjustmentBehavior` — `.automatic` (safe area), `.scrollableAxes`, `.never`, `.always`.
- `adjustedContentInset` — effective inset after safe area.
- `contentInset`/`verticalScrollIndicatorInsets`/`horizontalScrollIndicatorInsets`.
- `isPagingEnabled` — page-sized snapping.
- `decelerationRate` — `.normal` (0.998), `.fast` (0.99).
- `bounces`, `alwaysBounceVertical`, `alwaysBounceHorizontal`.
- `scrollsToTop` — tap status bar scrolls to top.
- `scrollViewDidScroll`, `scrollViewWillBeginDragging`, `scrollViewDidEndDragging(willDecelerate:)`, `scrollViewDidEndDecelerating`, `scrollViewWillEndDragging(_:withVelocity:targetContentOffset:)` — customize deceleration target.
- `viewForZooming`, `scrollViewDidZoom`, `minimumZoomScale`/`maximumZoomScale`.
- `contentLayoutGuide`/`frameLayoutGuide` — Auto Layout guides for scroll content.
- `UIRefreshControl` — `beginRefreshing`, `endRefreshing`, `isRefreshing`.
- `UIScrollView` keyboard avoidance: `keyboardDismissMode` (`.none`, `.interactive`, `.onDrag`).
- Nested scroll views: inner receives touches first; `delaysContentTouches`, `canCancelContentTouches`.

## 7. Appearance customization deep

### UIAppearance proxy (global)
```swift
UINavigationBar.appearance().tintColor = .systemBlue
UINavigationBar.appearance(whenContainedInInstancesOf: [MyNavController.self]).tintColor = .red
UITableViewCell.appearance().backgroundColor = .clear
UIButton.appearance(whenContainedInInstancesOf: [UIAlertController.self]).tintColor = .white
```

- `appearance()` — all instances; `appearance(whenContainedInInstancesOf:)` — scoped.
- `UIBarAppearance`/`UINavigationBarAppearance`/`UIToolbarAppearance`/`UITabBarAppearance` — modern bar styling: `backgroundColor`, `backgroundEffect`, `shadowColor`, `titleTextAttributes`, `largeTitleTextAttributes`, `backIndicatorImage`.
- `UISearchBarAppearance` — search bar styling.

### Trait overrides
```swift
view.traitOverrides.userInterfaceStyle = .dark
view.traitOverrides.horizontalSizeClass = .compact
view.traitOverrides.preferredContentSizeCategory = .accessibilityExtraExtraExtraLarge
```

- `traitOverrides` — per-view trait overrides without affecting children? — actually propagates to subtree unless overridden.
- `registerForTraitChanges([UITraitUserInterfaceStyle.self, UITraitHorizontalSizeClass.self]) { (self: Self, previousTraitCollection: UITraitCollection) in ... }` — iOS 17+ change callbacks.
- `UITraitCollection` properties: `userInterfaceStyle`, `userInterfaceLevel` (`.base`, `.elevated`, `.alert`), `horizontalSizeClass`, `verticalSizeClass`, `displayScale`, `displayGamut` (`.SRGB`, `.P3`), `legibilityWeight`, `accessibilityContrast` (`.normal`, `.high`), `layoutDirection`, `forceTouchCapability`, `hasDifferentiateWithoutColor`, `prefersCrossFadeTransitions`, `toolbarPlacement`, `userInterfaceIdiom`.
- `UIAccessibility` traits on `UIView`: `isAccessibilityElement`, `accessibilityLabel`, `accessibilityValue`, `accessibilityHint`, `accessibilityTraits` (`.button`, `.link`, `.header`, `.searchField`, `.image`, `.keyboardKey`, `.staticText`, `.summaryElement`, `.playsSound`, `.updatesFrequently`, `.startsMediaSession`, `.allowsDirectInteraction`, `.causesPageTurn`, `.tabBar`), `accessibilityElements` (custom children), `accessibilityElementIsHidden`, `accessibilityViewIsModal`, `accessibilityActivate()`, `accessibilityIncrement()`, `accessibilityDecrement()`.
- `UIAccessibility.post(notification: .layoutChanged/.screenChanged/.announcement, argument:)` — announce changes.
- `UIAccessibility.isVoiceOverRunning`, `.isBoldTextEnabled`, `.isReduceMotionEnabled`, `.isReduceTransparencyEnabled`, `.isDifferentiateWithoutColorEnabled`, `.isInvertColorsEnabled`, `.isGrayscaleEnabled`, `.isClosedCaptioningEnabled`, `.isGuidedAccessEnabled`, `.isSpeakSelectionEnabled`, `.isSpeakScreenEnabled`, `.isSwitchControlRunning`, `.isAssistiveTouchRunning`, `.isShakeToUndoEnabled`, `.isOnOffSwitchLabelsEnabled`, `.isVideoAutoplayEnabled`, `.isAudioDescriptionEnabled`, `.isMonoAudioEnabled`, `.isSpatialAudioEnabled`? — check current list.

## 8. Keyboard & input deep

- `inputAccessoryView` — custom toolbar above keyboard: `UITextField.inputAccessoryView`, `UITextView.inputAccessoryView`; toolbar with Done/Next buttons.
- `inputView` — custom input view (replaces keyboard): `UITextField.inputView`, `UITextView.inputView`; `inputViewController` for custom keyboard VCs.
- `reloadInputViews()` — refresh input views after changes.
- `becomeFirstResponder()`/`resignFirstResponder()` — focus management; `isFirstResponder`.
- Keyboard notifications: `UIResponder.keyboardWillShowNotification`, `.keyboardDidShow`, `.keyboardWillHide`, `.keyboardDidHide`, `.keyboardWillChangeFrame`, `.keyboardDidChangeFrame` — userInfo: `UIResponder.keyboardFrameEndUserInfoKey`, `.animationDurationUserInfoKey`, `.animationCurveUserInfoKey`.
- `keyboardLayoutGuide` — Auto Layout guide tracking keyboard (iOS 15+): `view.keyboardLayoutGuide.topAnchor`.
- `UITextInput` protocol — `beginningOfDocument`, `endOfDocument`, `position(from:in:offset:)`, `position(from:in:offset:)`, `compare`, `offset(from:to:)`, `text(in:)`, `replace(_:withText:)`, `setMarkedText`, `unmarkText`, `selectedTextRange`, `markedTextRange`, `firstRect(for:)`.
- `UITextDocumentProxy` — for custom keyboards: `documentContextBeforeInput`, `documentContextAfterInput`, `documentIdentifier`, `hasText`, `adjustTextPosition`, `setMarkedText`, `insertText`, `deleteBackward`.
- `UIKeyCommand` — hardware keyboard: `UIKeyCommand(input:modifierFlags:action:)`, `keyCommands` property on VC/Responder; `discoverabilityTitle` for HUD; `wantsPriorityOverSystemBehavior`.
- `UIKeyboardHIDUsage` — special keys: `.keyboardReturnOrEnter`, `.keyboardEscape`, `.keyboardDeleteOrBackspace`, `.keyboardTab`, `.keyboardSpacebar`, `.keyboardLeftArrow` etc.
- `UITextInputMode`/`UITextInputMode.activeInputModes` — keyboard modes.

## 9. Drag & drop deep

```swift
// Source
let interaction = UIDragInteraction(delegate: self)
view.addInteraction(interaction)

func dragInteraction(_ interaction: UIDragInteraction, itemsForBeginning session: UIDragSession) -> [UIDragItem] {
  let item = UIDragItem(itemProvider: NSItemProvider(object: note.title as NSString))
  item.localObject = note   // for local drops
  return [item]
}

// Preview
func dragInteraction(_ interaction: UIDragInteraction, previewForLifting item: UIDragItem,
  session: UIDragSession) -> UITargetedDragPreview? {
  let params = UIDragPreviewParameters()
  params.backgroundColor = .clear
  params.visiblePath = UIBezierPath(roundedRect: cell.bounds, cornerRadius: 12)
  return UITargetedDragPreview(view: cell, parameters: params)
}

// Destination
let dropInteraction = UIDropInteraction(delegate: self)
view.addInteraction(dropInteraction)

func dropInteraction(_ interaction: UIDropInteraction, canHandle session: UIDropSession) -> Bool {
  session.canLoadObjects(ofClass: Note.self)
}
func dropInteraction(_ interaction: UIDropInteraction, sessionDidUpdate session: UIDropSession) -> UIDropProposal {
  UIDropProposal(operation: .copy)
}
func dropInteraction(_ interaction: UIDropInteraction, performDrop session: UIDropSession) {
  for item in session.items {
    _ = item.loadObject(ofClass: Note.self) { note, error in ... }
  }
}
```

- `UIDragInteraction`/`UIDropInteraction` — add via `view.addInteraction`.
- `UIDragSession`/`UIDropSession` — `items`, `location(in:)`, `hasItemsConforming`, `canLoadObjects`, `loadObjects`, `localDragSession`.
- `UIDragItem` — `itemProvider` (NSItemProvider), `localObject`, `previewProvider`.
- `NSItemProvider` — `registerObject`, `loadObject`, `loadFileRepresentation`, `loadDataRepresentation`, `hasItemConformingToTypeIdentifier`.
- `UTType` — `.plainText`, `.url`, `.image`, `.fileURL`, `.data`, `.json`, `.pdf`, `.png`, `.jpeg`, custom.
- `UIDropProposal` — `operation` (`.copy`, `.move`, `.generic`, `.cancel`, `.forbidden`), `dropIsPrecise`, `prefersFullSizePreview`.
- `UIDragPreviewParameters` — `backgroundColor`, `visiblePath`, `shadowPath`.
- `UITargetedDragPreview`/`UITargetedPreview` — custom drag previews.
- Spring loading: `UIDropInteraction` `springLoadingHighlight`? — `UIDropInteractionDelegate` `dropInteraction(_:sessionDidUpdate:)` return proposal with `prefersFullSizePreview`.
- Drag between apps: pasteboard-backed; within app: `localObject` fast path.

## 10. Modern cell APIs

```swift
// Cell registration (iOS 14+)
let registration = UITableView.CellRegistration<MyCell, Item> { cell, indexPath, item in
  cell.textLabel?.text = item.name
  cell.accessoryType = .disclosureIndicator
}
tableView.dequeueConfiguredReusableCell(using: registration, for: indexPath, item: item)

// Content configuration
var config = cell.defaultContentConfiguration()
config.text = item.name
config.secondaryText = item.subtitle
config.image = UIImage(systemName: "doc")
config.imageProperties.tintColor = .systemBlue
config.textProperties.color = .label
config.secondaryTextProperties.color = .secondaryLabel
cell.contentConfiguration = config

// Background configuration
var bgConfig = UIBackgroundConfiguration.listCell()
bgConfig.backgroundColor = .secondarySystemBackground
bgConfig.cornerRadius = 12
cell.backgroundConfiguration = bgConfig
```

- `UIListContentConfiguration` — `text`, `secondaryText`, `image`, `secondaryImage`, `tertiaryText`, `quaternaryText`, `attributedText`, `textProperties`, `secondaryTextProperties`, `imageProperties`, `secondaryImageProperties`, `axes` (`.vertical`, `.horizontal`), `directionalLayoutMargins`.
- `UIBackgroundConfiguration` — `backgroundColor`, `backgroundVisualEffect`, `cornerRadius`, `margins`, `backgroundInsets`, `strokeColor`, `strokeWidth`, `strokeCornerRadius`.
- `UICellConfigurationState` — `isSelected`, `isHighlighted`, `isFocused`, `isEditing`, `showsReorderControl`, `isSwiped` — for state-based config.
- `UITableViewDiffableDataSource` — `apply(snapshot, animatingDifferences:)`; `NSDiffableDataSourceSnapshot` with `appendSections`, `appendItems`, `deleteSections`, `deleteItems`, `moveSection`, `moveItem`, `reloadSections`, `reloadItems`, `reconfigureItems`.
- `UICollectionView.CellRegistration`/`SupplementaryRegistration` — modern cell/supplementary registration.
- `UICollectionViewDiffableDataSource` — same snapshot API.

## 11. Performance deep

- **Main thread**: all UI on main; heavy work → `DispatchQueue.global().async` or `Task.detached`; hop back `DispatchQueue.main.async`/`@MainActor`.
- **Cell reuse**: `prepareForReuse` reset state; configure in `cellForRow` not `init`; avoid expensive work in `layoutSubviews`.
- **Image downsampling**: don't decode full-res into thumbnails — `UIImage.preparingForDisplay()` or ImageIO `CGImageSourceCreateThumbnailAtIndex`.
- **Prefetching**: `UICollectionViewDataSourcePrefetching`/`UITableViewDataSourcePrefetching` — preload data/images before display.
- **Auto Layout cost**: complex hierarchies expensive; flatten, use `UIStackView`, avoid constraint churn.
- **Layer effects**: `shouldRasterize` for complex static content; `drawsAsynchronously` for expensive drawing; `cornerRadius` + `masksToBounds` = offscreen render — use `layer.cornerCurve` + `cornerRadius` carefully or `CAShapeLayer` mask.
- **Instruments**: Time Profiler, Allocations, Leaks, Core Animation, Animation Hitches, Hangs, os_signpost.
- **Memory warnings**: `didReceiveMemoryWarning` — release caches, images; `NSCache` auto-evicts.
- **Launch**: `didFinishLaunching` fast; defer init; `os_signpost` + `XCTApplicationLaunchMetric`.
- **Scroll hitches**: avoid work in `scrollViewDidScroll`; prefetch; `CADisplayLink` for custom animation.

## 12. Modern UIKit (iOS 17-27)

- `UIContentUnavailableConfiguration` — empty states: `.search()`, `.noContent()`, custom.
- `UISheetPresentationController` — `.medium()/.large()` detents, `.largestUndimmed`, custom detents, grabber.
- `UIUpdateLink` (iOS 18+) — display-synced updates replacing `CADisplayLink` for SwiftUI-adjacent work.
- `UIButton.Configuration` — `.plain/.gray/.tinted/.filled/.bordered/.borderedProminent/.borderless`, `cornerStyle`, `image+title+subtitle`, `configurationUpdateHandler`.
- `UIMainMenuSystem`/`UIMenuBuilder` — keyboard shortcut + menu building.
- `UISearchTab`/`UISearchTabPlacement` — iOS 26+ search in tab bars.
- `UIListContentConfiguration`/`UIBackgroundConfiguration` — cell content/background config.
- `UIHoverGestureRecognizer`, `UIPointerStyle`, `UIPointerInteraction` — pointer/pencil.
- `UIViewController` `updateProperties()`/`viewIsAppearing(_:)` — iOS 17+ lifecycle.
- `UITraitCollection` `accessibilityReduceMotion` etc.
- `UIKeyCommand` `.discoverabilityTitle` for keyboard shortcut HUD.
- `UIActivityItemsConfiguration` — async share items.
- `UICollectionView` compositional layout improvements.
- `UITableView` self-sizing improvements.
- `UIWindowScene` `effectiveGeometry` — window geometry including safe areas.
- `UIApplicationDelegate` `application(_:configurationForConnecting:options:)` for scene setup.
- `UIGlassEffect` (iOS 26+) — Liquid Glass effect views.
- `UIUpdateLink` — frame-synced updates.
- `UINavigationBarAppearance`/`UIToolbarAppearance`/`UITabBarAppearance` — consistent bar styling.
- `UIScrollView` `topEdgeEffect`/`bottomEdgeEffect` — scroll edge effects.
- `UIView` `tintColorDidChange`, `override func tintColorDidChange()`.
- `UIWindow` `windowScene` — scene-aware windows.
- `UIViewController` `preferredScreenEdgesDeferringSystemGestures` — edge gesture deferral.
- `UIViewController` `preferredStatusBarStyle`/`prefersHomeIndicatorAutoHidden`/`prefersStatusBarHidden` — status bar control.
- `UIViewController` `modalInPresentation` — prevent swipe-to-dismiss.
- `UIViewController` `isModalInPresentation` — same.
- `UIViewController` `definesPresentationContext` — presentation context.
- `UIViewController` `providesPresentationContextTransitionStyle` — transition style.
- `UIViewController` `modalTransitionStyle` — coverVertical, flipHorizontal, crossDissolve, partialCurl.
- `UIViewController` `modalPresentationCapturesStatusBarAppearance` — status bar during modal.

## 13. Common advanced UIKit bugs → fixes

| Bug | Cause | Fix |
|---|---|---|
| Text not wrapping | `numberOfLines = 1`, missing width constraint | set `numberOfLines = 0`, add width constraint |
| Custom transition jerky | wrong container view, missing layout | use `context.containerView`, call `layoutIfNeeded` |
| Compositional layout crash | invalid size dimensions, group/item mismatch | check `NSCollectionLayoutSize` dimensions |
| Diffable data source crash | duplicate IDs, invalid snapshot ops | ensure unique IDs, check section/item existence |
| Auto Layout ambiguous | missing constraints, conflicting priorities | add required constraints, adjust priorities |
| Scroll view not scrolling | contentSize zero, constraints missing | set contentSize or use Auto Layout with contentLayoutGuide |
| Gesture not firing | gesture recognizer delegate returning false, view not hit-testable | check delegate, `isUserInteractionEnabled`, `hitTest` |
| Keyboard covers input | no scroll adjustment, safe area ignored | use `keyboardLayoutGuide`, adjust contentInset |
| Cell reuse stale data | not resetting in `prepareForReuse` | reset all state in `prepareForReuse` |
| Image blurry | no downsampling, wrong contentMode | downsample, set `contentMode = .scaleAspectFill` + `clipsToBounds` |
| Memory spike in scroll | images not cached, cells not reused | use `NSCache`, reuse cells, downsample |
| Dark mode wrong | hardcoded colors | use dynamic colors, asset catalog |
| Dynamic Type broken | fixed font sizes | use `UIFont.preferredFont(forTextStyle:)` + `UIFontMetrics` |
| RTL layout broken | left/right constraints | use leading/trailing, `semanticContentAttribute` |
| Snapshot blank | `drawHierarchy` timing, layer not rendered | use `layer.render(in:)` for layer snapshot, `drawHierarchy` for view |
| Accessibility broken | missing labels, wrong traits | add `accessibilityLabel`/`Traits`, test with VoiceOver |
