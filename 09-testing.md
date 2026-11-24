# Testing Reference

XCTest (unit + UI), Swift Testing (modern), snapshot testing, mocking, performance tests, CI integration.

## 1. XCTest — unit tests

```swift
import XCTest
@testable import MyApp   // @testable exposes internal symbols

final class UserTests: XCTestCase {
  var user: User!
  override func setUp() { user = User(name: "Alice", age: 30) }   // before each test
  override func tearDown() { user = nil }                          // after each test

  func testName() {
    XCTAssertEqual(user.name, "Alice")
    XCTAssertTrue(user.isAdult)
    XCTAssertNil(user.nickname)
    XCTAssertNotNil(user.email)
    XCTAssertGreaterThan(user.age, 18)
    XCTAssertEqual(user.name, "Alice", "name should be Alice")   // with message
  }

  func testThrows() {
    XCTAssertThrowsError(try user.validate()) { error in
      XCTAssertEqual(error as? ValidationError, .invalidAge)
    }
    XCTAssertNoThrow(try User(name: "Bob", age: 25).validate())
  }

  func testPerformance() {
    measure(metrics: [XCTClockMetric()]) {
      _ = heavyComputation()
    }
  }
}
```

- Assertions: `XCTAssertEqual`, `XCTAssertNotEqual`, `XCTAssertTrue`, `XCTAssertFalse`, `XCTAssertNil`, `XCTAssertNotNil`, `XCTAssertGreaterThan`, `XCTAssertLessThan`, `XCTAssertGreaterThanOrEqual`, `XCTAssertLessThanOrEqual`, `XCTAssertThrowsError`, `XCTAssertNoThrow`, `XCTFail`, `XCTAssertEqualWithAccuracy` (floats), `XCTAssertIdentical`/`XCTAssertNotIdentical` (identity).
- `XCTUnwrap` — unwrap optionals in tests: `let user = try XCTUnwrap(optionalUser)`.
- Async tests: `func testAsync() async throws` — XCTest supports async test methods natively.
- Expectations: `let exp = expectation(description: "callback")`, `waitForExpectations(timeout:handler:)` — for callbacks/delegates.
- `measure(block:)` — performance measurement; `XCTMetric` types: `XCTClockMetric`, `XCTCPUMetric`, `XCTMemoryMetric`, `XCTStorageMetric`, `XCTApplicationLaunchMetric`, `XCTOSSignpostMetric`, `XCTAnimationHitchMetric` (UI tests).
- `XCTAttachment` — attach screenshots/logs/data to test results: `add(XCTAttachment(screenshot:))`.
- `setUpWithError`/`tearDownWithError` — throwing setup/teardown.
- `continueAfterFailure = false` — stop test on first failure.
- Test plans: `.xctestplan` JSON — configure test targets, parallelization, code coverage, environment variables; multiple plans for different configs.
- `XCTExpectFailure` — expected failures (known bugs).
- `XCTSkip`/`XCTSkipUnless`/`XCTSkipIf` — conditional skipping.
- `XCTContext.runActivity` — organize test output into activities.
- Parameterized tests: loop or use `for` in test method; Swift Testing has native parameterization.

## 2. Swift Testing (modern, iOS 18+/macOS 15+)

```swift
import Testing
@testable import MyApp

struct UserTests {
  @Test func adultUser() {
    let user = User(name: "Alice", age: 30)
    #expect(user.isAdult)
    #expect(user.name == "Alice")
    #expect(user.age > 18)
  }

  @Test func invalidAgeThrows() throws {
    let user = User(name: "Bob", age: 15)
    #expect(throws: ValidationError.self) {
      try user.validate()
    }
    #expect(throws: ValidationError.invalidAge) { try user.validate() }  // specific error
  }

  @Test(arguments: [18, 21, 30, 65]) func ageValidation(age: Int) {
    let user = User(name: "Test", age: age)
    #expect(user.isAdult)
  }

  @Test(.tags(.critical)) func criticalPath() { ... }

  @Test(.timeLimit(.minutes(1))) func slowTest() async throws { ... }

  @Test(.enabled(if: ProcessInfo.processInfo.environment["CI"] != nil))
  func ciOnlyTest() { ... }

  @Test(.disabled("broken")) func knownFailing() { ... }

  @Suite(.serialized) struct SerializedSuite { ... }   // run tests serially
  @Suite struct GroupedTests { ... }
}
```

- `@Test` — test function/struct method; supports `async`, `throws`, parameterized via `arguments:`.
- `#expect` — assertion macro: `#expect(condition)`, `#expect(a == b)`, `#expect(throws:)` for error checking.
- `#require` — like expect but throws on failure (for setup/unwrapping): `let user = try #require(optional)`.
- `#expect(throws: SomeError.self)` / `#expect(throws: SpecificError.case)` — error assertions.
- Parameterized: `@Test(arguments: [1,2,3])`, `@Test(arguments: zip(names, ages))`, collections, sequences.
- `@Suite` — group tests; `.serialized` trait for serial execution; nested suites.
- Traits: `.tags(.TagName)`, `.timeLimit(.seconds/.minutes)`, `.enabled(if:)`, `.disabled("reason")`, `.bug("id")`, `.serialized`.
- `Confirmation` — for async callbacks: `let confirmation = Confirmation()`, `confirmation.expect()`, `await confirmation.wait()`.
- `TestClock`? — no; use `ContinuousClock`/`SuspendingClock` for time-based tests.
- Custom traits: `Trait`, `SuiteTrait`, `TestTrait` protocols.
- Migration: XCTest and Swift Testing coexist; run both in same target.
- `import Testing` requires iOS 18+/macOS 15+ SDK; back-deploys? — Swift Testing requires newer OS for some features; check deployment target.

## 3. UI Testing (XCUITest)

```swift
import XCTest

final class LoginUITests: XCTestCase {
  var app: XCUIApplication!

  override func setUpWithError() throws {
    continueAfterFailure = false
    app = XCUIApplication()
    app.launchArguments = ["-UITestMode", "YES"]   // app reads these
    app.launchEnvironment = ["TEST_USER": "test@example.com"]
    app.launch()
  }

  func testLoginFlow() throws {
    let emailField = app.textFields["emailTextField"]
    XCTAssertTrue(emailField.waitForExistence(timeout: 5))
    emailField.tap()
    emailField.typeText("test@example.com")

    let passwordField = app.secureTextFields["passwordField"]
    passwordField.tap()
    passwordField.typeText("password123")

    app.buttons["loginButton"].tap()

    // Wait for navigation
    let welcomeText = app.staticTexts["Welcome, Test User"]
    XCTAssertTrue(welcomeText.waitForExistence(timeout: 10))
  }

  func testEmptyEmailValidation() {
    app.buttons["loginButton"].tap()
    XCTAssertTrue(app.staticTexts["Email is required"].exists)
  }
}
```

- `XCUIApplication` — `launch()`, `launchArguments`, `launchEnvironment`, `terminate()`, `state` (`.runningForeground`, `.runningBackground`, `.notRunning`), `windows`, `alerts`, `keyboards`, `activityIndicators`, `scrollViews`, `tables`, `collectionViews`, `webViews`, `images`, `buttons`, `staticTexts`, `textFields`, `secureTextFields`, `switches`, `sliders`, `steppers`, `segmentedControls`, `pickers`, `datePickers`, `searchFields`, `links`, `cells`, `otherElements`, `tabBars`, `toolbars`, `navigationBars`, `statusBars`.
- Element queries: `app.buttons["label"]`, `app.buttons.element(boundBy: 0)`, `app.descendants(matching: .button).matching(identifier: "x")`, `app.buttons.matching(NSPredicate(format: "label CONTAINS 'x'"))`.
- `XCUIElement` — `tap()`, `doubleTap()`, `press(forDuration:)`, `typeText()`, `swipeLeft/Right/Up/Down`, `pinch(withScale:velocity:)`, `rotate(_:withVelocity:)`, `scroll()`, `adjust(toNormalizedSliderPosition:)`, `isHittable`, `exists`, `isEnabled`, `isSelected`, `hasFocus`, `frame`, `label`, `value`, `placeholderValue`, `identifier`, `elementType`, `waitForExistence(timeout:)`.
- `XCUIElementQuery` — `children(matching:)`, `descendants(matching:)`, `matching(identifier:)`, `matching(NSPredicate)`, `element(boundBy:)`, `firstMatch`, `allElementsBoundByIndex`, `count`.
- Accessibility identifiers: set `accessibilityIdentifier` on views for stable UI test queries — NEVER use localized labels in tests.
- `XCUIDevice` — `shared.orientation`, `shared.press(.home)`, `shared.siriService`.
- `XCUIRemote` — control other devices.
- Screenshots: `XCTAttachment(screenshot: XCUIScreen.main.screenshot())`, `app.screenshot()`.
- `XCUIApplication().launchArguments` — app reads `ProcessInfo.processInfo.arguments` or `UserDefaults` for test mode.
- Performance UI tests: `measure(metrics: [XCTOSSignpostMetric(), XCTApplicationLaunchMetric(), XCTAnimationHitchMetric(application: app)])`.
- Accessibility audit: `try app.performAccessibilityAudit()` (iOS 17+) — audits for common issues.
- `XCUIElement.typeText` — requires keyboard; for secure fields, use `typeText` with app keyboard or paste via `UIPasteboard`.
- Keyboard handling: `app.keyboards.buttons["return"].tap()`, `app.keyboards.keys["a"].tap()`.
- Scroll: `app.scrollViews.firstMatch.swipeUp()`, `app.tables.firstMatch.cells.element(boundBy:).swipeLeft()`.
- Alerts: `app.alerts["title"].buttons["OK"].tap()`, `app.sheets`.
- Deep links: `app.open(URL(string: "myapp://path")!)` or launch with `XCUIApplication.open`.
- Push notifications: `XCUIApplication.pushNotification`? — simulate via `simctl push` or test deep link handling.
- App state: `XCUIDevice.shared.press(.home)` for backgrounding.

## 4. Snapshot testing

### swift-snapshot-testing (Point-Free)
```swift
import SnapshotTesting

func testViewSnapshot() {
  let view = MyView()
  assertSnapshot(of: view, as: .image(layout: .fixed(width: 390, height: 844)))
}
// Strategies: .image, .image(precision:), .image(layout: .device(.iPhone13)), .image(layout: .sizeThatFits),
//   .dump, .recursiveDescription, .dump(options:), .hierarchy, .hierarchy(precision:), .traits
```
- Records reference images in `__Snapshots__` directory; fails on diff; `record` mode to update.
- `SnapshotTesting` supports SwiftUI views (`View`), UIKit views, `UIViewController`, `NSView`, `NSViewController`.
- Layouts: `.fixed(width:height:)`, `.sizeThatFits`, `.device(.iPhone13)`/`.device(.iPadPro)`.
- Precision: `.image(precision: 0.99)` for anti-aliasing tolerance.

### iOSSnapshotTestCase (legacy)
```swift
import FBSnapshotTestCase
class MyTests: FBSnapshotTestCase {
  func testSnapshot() {
    let view = MyView(frame: CGRect(x: 0, y: 0, width: 375, height: 667))
    FBSnapshotTestCase.verifyView(view)
  }
}
```
- Older library; swift-snapshot-testing preferred.

## 5. Mocking & dependency injection

### Protocol-based mocking (preferred)
```swift
protocol UserRepository {
  func fetchUser(id: UUID) async throws -> User
}

struct MockUserRepository: UserRepository {
  var users: [UUID: User] = [:]
  var error: Error?
  func fetchUser(id: UUID) async throws -> User {
    if let error { throw error }
    return users[id]!
  }
}

@Test func testViewModel() async throws {
  let repo = MockUserRepository(users: [id: user])
  let vm = UserViewModel(repository: repo)
  await vm.load()
  #expect(vm.user?.name == "Alice")
}
```

- Protocol + hand-written mock — simplest, most flexible.
- `Mockolo` — generates mocks from protocols: `@mockable protocol` annotation.
- `Mockingbird` — Swift mocking framework: `@Mock`, `when`, `verify`.
- `Sourcery` — code generation for mocks, boilerplate.
- `Factory`/`Swinject` — DI containers for production + test injection.

### Dependency injection patterns
```swift
// Constructor injection (preferred)
struct UserViewModel {
  let repository: UserRepository
  let analytics: Analytics
  init(repository: UserRepository, analytics: Analytics) { ... }
}

// Environment injection (SwiftUI)
@Observable final class AppDependencies {
  let apiClient: APIClient
  let repository: UserRepository
  init(apiClient: APIClient) { self.apiClient = apiClient; repository = UserRepository(api: apiClient) }
}
// In App:
@State private var deps = AppDependencies(apiClient: APIClient())
ContentView().environment(deps)
// In View:
@Environment(AppDependencies.self) private var deps
```

## 6. Testing patterns & best practices

### Arrange-Act-Assert
```swift
@Test func calculateTotal() {
  // Arrange
  let cart = Cart(items: [Item(price: 10), Item(price: 20)])
  // Act
  let total = cart.total
  // Assert
  #expect(total == 30)
}
```

### Given-When-Then
```swift
@Test func loginSuccess() async throws {
  // Given
  let auth = MockAuthService(credentials: validCredentials)
  let vm = LoginViewModel(auth: auth)
  // When
  try await vm.login()
  // Then
  #expect(vm.isLoggedIn)
}
```

### Test naming
- `test<Method>_<Scenario>_<ExpectedResult>` — XCTest convention.
- `@Test("description") func testName()` — Swift Testing supports descriptions.
- Describe behavior, not implementation: `testLoginWithInvalidPasswordShowsError` not `testLoginMethod`.

### What to test
- **Unit**: business logic, models, view models, utilities, edge cases, error paths.
- **Integration**: API client + decoder, database + repository, multiple components together.
- **UI**: critical user flows (login, checkout, onboarding), not every screen.
- **Snapshot**: view appearance, design system components, complex layouts.
- **Performance**: launch time, scroll performance, memory usage, expensive computations.

### What NOT to test
- Framework code (SwiftUI internals, UIKit behavior).
- Trivial getters/setters.
- Implementation details that change frequently.
- Every possible input — focus on edge cases and boundaries.

## 7. Performance testing

```swift
func testScrollPerformance() {
  let app = XCUIApplication()
  app.launch()
  measure(metrics: [XCTOSSignpostMetric(), XCTAnimationHitchMetric(application: app), XCTCPUMetric(), XCTMemoryMetric()]) {
    app.scrollViews.firstMatch.swipeUp()
    app.scrollViews.firstMatch.swipeDown()
  }
}

func testLaunchPerformance() {
  measure(metrics: [XCTApplicationLaunchMetric()]) {
    XCUIApplication().launch()
  }
}
```

- `XCTClockMetric` — wall-clock time.
- `XCTCPUMetric` — CPU usage.
- `XCTMemoryMetric` — memory footprint.
- `XCTStorageMetric` — disk I/O.
- `XCTApplicationLaunchMetric` — app launch time (cold/warm).
- `XCTOSSignpostMetric` — os_signpost intervals.
- `XCTAnimationHitchMetric` — dropped frames during animation.
- `XCTMemoryMetric(application:)` — app memory in UI tests.
- `XCTCPUMetric(application:)` — app CPU in UI tests.

## 8. Test plans & CI

### Test plans (.xctestplan)
```json
{
  "configurations": [{
    "id": "abc123",
    "name": "Full Test",
    "options": {
      "testTimeoutsEnabled": true,
      "defaultTestExecutionTimeAllowance": 600,
      "testRepetitionMode": "retryOnFailure"
    }
  }],
  "testTargets": [{
    "target": { "containerPath": "container:MyApp.xcodeproj", "identifier": "...", "name": "MyAppTests" },
    "parallelizable": true,
    "selectedTests": ["MyAppTests"],
    "skippedTests": []
  }],
  "version": 1
}
```

- Multiple plans: unit only, UI only, full suite, smoke tests.
- Parallelization: `parallelizable: true` + `parallelExecutionEnabled` — splits across simulators.
- Code coverage: `codeCoverageEnabled: true` in plan or scheme.

### xcodebuild test
```bash
# Run all tests
xcodebuild test -scheme MyApp -destination 'platform=iOS Simulator,name=iPhone 17' \
  -resultBundlePath TestResults.xcresult

# Specific test plan
xcodebuild test -scheme MyApp -testPlan Full -destination 'platform=iOS Simulator,name=iPhone 17'

# Specific tests only
xcodebuild test -scheme MyApp -only-testing:MyAppTests/UserTests -destination 'platform=iOS Simulator,name=iPhone 17'

# UI tests only
xcodebuild test -scheme MyApp -only-testing:MyAppUITests -destination 'platform=iOS Simulator,name=iPhone 17'

# Code coverage
xcodebuild test -scheme MyApp -enableCodeCoverage YES -resultBundlePath TestResults.xcresult

# Export results
xcrun xcresulttool get test-results summary --path TestResults.xcresult
xcrun xcresulttool get test-results tests --path TestResults.xcresult
xcrun xcresulttool get test-results test-details --path TestResults.xcresult --test-id <id>
```

- `xcresulttool` — parse `.xcresult` bundles for CI reporting.
- `xcodebuild -resultBundlePath` — saves test results for later inspection.
- `-parallel-testing-enabled YES` — parallel UI tests across simulators.
- `-maximum-parallel-testing-workers N` — limit parallel workers.

### Xcode Cloud
- Built-in CI: connects to git repo, runs tests on every push/PR.
- `ci_scripts/ci_post_clone.sh` — custom setup script.
- `ci_scripts/ci_pre_xcodebuild.sh` — pre-build script.
- `ci_scripts/ci_post_xcodebuild.sh` — post-build script.
- Environment variables: `CI`, `CI_BUILD_ID`, `CI_COMMIT`, `CI_BRANCH`, `CI_TAG`.
- Test results visible in App Store Connect.

### Fastlane
```ruby
# Fastfile
lane :test do
  scan(scheme: "MyApp", device: "iPhone 17", clean: true)
end
```
- `scan` — run tests; `snapshot` — UI screenshot automation; `frameit` — video frames; `deliver` — App Store metadata; `pilot` — TestFlight.

## 9. Common testing bugs → fixes

| Bug | Cause | Fix |
|---|---|---|
| Test flaky (passes sometimes) | timing, shared state, network | use expectations, mock network, isolate tests, `.serialized` suite |
| UI test can't find element | wrong identifier, element not visible, animation | use `accessibilityIdentifier`, `waitForExistence`, scroll to element |
| Async test hangs | expectation never fulfilled, await never returns | check async code path, add timeout, verify mock returns |
| Snapshot test fails on CI | different OS/font rendering | use `.image(precision:)` tolerance, consistent simulator OS |
| Test crashes on launch | missing test configuration, force unwrap | use `XCTUnwrap`, guard test setup, check environment |
| Memory leak in test | retain cycle in test object | use `weak`/`unowned`, check with Instruments Leaks |
| Coverage low | tests only happy path | add edge cases, error paths, boundary conditions |
| UI test slow | real network calls, unnecessary waits | mock API, use launch arguments for test mode, reduce timeouts |
| Swift Testing not discovered | wrong target membership, missing import | ensure test file in test target, `import Testing`, iOS 18+ deployment |

## 10. Testing checklist for new features

- [ ] Unit tests for business logic (models, view models, services).
- [ ] Edge cases: empty input, nil values, boundary conditions, error states.
- [ ] Async paths: success, failure, cancellation, timeout.
- [ ] Integration test for API/database layer.
- [ ] UI test for critical user flow (if applicable).
- [ ] Snapshot test for new views/components.
- [ ] Performance test for expensive operations.
- [ ] Accessibility audit (XCUITest `performAccessibilityAudit`).
- [ ] Localization test (pseudo-localization, RTL).
- [ ] Dark mode / Dynamic Type test.
- [ ] Offline behavior test.
- [ ] Memory leak check (Instruments).
