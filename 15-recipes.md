# Recipes — Copy-Paste Patterns & Checklists

Practical patterns for common Apple-platform tasks. Adapt to project conventions.

## 1. New SwiftUI app skeleton

```swift
// MyApp.swift
import SwiftUI
import SwiftData

@main
struct MyApp: App {
  @State private var appStore = AppStore()
  @State private var deps = AppDependencies()

  var body: some Scene {
    WindowGroup {
      ContentView()
        .environment(appStore)
        .environment(deps)
        .task { await appStore.load() }
    }
    .modelContainer(for: [Note.self])   // SwiftData
    Settings { SettingsView() }          // macOS Cmd+,
    MenuBarExtra("Widget", systemImage: "star") { MenuContent() }  // optional
  }
}

// ContentView.swift
struct ContentView: View {
  @Environment(AppStore.self) private var store
  @Environment(AppDependencies.self) private var deps
  @State private var searchText = ""

  var body: some View {
    NavigationStack {
      List(store.items) { item in
        NavigationLink(value: item) { ItemRow(item: item) }
      }
      .searchable(text: $searchText)
      .navigationTitle("Items")
      .navigationDestination(for: Item.self) { item in DetailView(item: item) }
      .toolbar {
        ToolbarItem(placement: .primaryAction) {
          Button { store.addItem() } label: { Image(systemName: "plus") }
        }
      }
      .refreshable { await store.refresh() }
    }
  }
}
```

## 2. Networking layer (actor + async)

```swift
import Foundation

enum APIError: Error, LocalizedError {
  case invalidURL, invalidResponse, httpError(Int), decodingError(Error)
  var errorDescription: String? {
    switch self {
    case .invalidURL: return "Invalid URL"
    case .invalidResponse: return "Invalid response"
    case .httpError(let code): return "HTTP error: \(code)"
    case .decodingError(let error): return "Decoding error: \(error.localizedDescription)"
    }
  }
}

actor APIClient {
  static let shared = APIClient()
  private let session: URLSession
  private let decoder: JSONDecoder = {
    let d = JSONDecoder()
    d.keyDecodingStrategy = .convertFromSnakeCase
    d.dateDecodingStrategy = .iso8601
    return d
  }()

  init(session: URLSession = .shared) { self.session = session }

  func get<T: Decodable>(_ path: String, query: [String: String]? = nil) async throws -> T {
    guard var comps = URLComponents(string: "https://api.example.com") else { throw APIError.invalidURL }
    comps.path = path
    comps.queryItems = query?.map { URLQueryItem(name: $0.key, value: $0.value) }
    guard let url = comps.url else { throw APIError.invalidURL }

    var request = URLRequest(url: url)
    request.setValue("application/json", forHTTPHeaderField: "Accept")
    request.setValue("Bearer \(await AuthManager.shared.token ?? "")", forHTTPHeaderField: "Authorization")

    let (data, response) = try await session.data(for: request)
    guard let http = response as? HTTPURLResponse else { throw APIError.invalidResponse }
    guard (200..<300).contains(http.statusCode) else { throw APIError.httpError(http.statusCode) }
    do { return try decoder.decode(T.self, from: data) }
    catch { throw APIError.decodingError(error) }
  }

  func post<T: Decodable, B: Encodable>(_ path: String, body: B) async throws -> T {
    guard let url = URL(string: "https://api.example.com\(path)") else { throw APIError.invalidURL }
    var request = URLRequest(url: url)
    request.httpMethod = "POST"
    request.setValue("application/json", forHTTPHeaderField: "Content-Type")
    request.httpBody = try JSONEncoder().encode(body)
    let (data, response) = try await session.data(for: request)
    guard let http = response as? HTTPURLResponse, (200..<300).contains(http.statusCode) else {
      throw APIError.httpError((response as? HTTPURLResponse)?.statusCode ?? -1)
    }
    return try decoder.decode(T.self, from: data)
  }
}

// Usage
let users: [User] = try await APIClient.shared.get("/users", query: ["limit": "10"])
```

## 3. Dependency injection (environment)

```swift
@Observable
final class AppDependencies {
  let apiClient: APIClient
  let authManager: AuthManager
  let repository: UserRepository

  init(apiClient: APIClient = .shared, authManager: AuthManager = .shared) {
    self.apiClient = apiClient
    self.authManager = authManager
    self.repository = UserRepository(api: apiClient)
  }
}

@main struct MyApp: App {
  @State private var deps = AppDependencies()
  var body: some Scene {
    WindowGroup { ContentView().environment(deps) }
  }
}

// In view:
@Environment(AppDependencies.self) private var deps
```

## 4. SwiftData repository pattern

```swift
@Model
final class Note {
  var title: String = ""
  var body: String = ""
  var createdAt: Date = Date()
  @Relationship(deleteRule: .cascade) var tags: [Tag]? = []
  init(title: String, body: String = "") { self.title = title; self.body = body }
}

@Model
final class Tag {
  var name: String = ""
  @Relationship(inverse: \Note.tags) var notes: [Note]? = []
}

actor NoteRepository {
  private let context: ModelContext
  init(context: ModelContext) { self.context = context }

  func allNotes() throws -> [Note] {
    try context.fetch(FetchDescriptor<Note>(sortBy: [SortDescriptor(\.createdAt, order: .reverse)]))
  }
  func notes(matching query: String) throws -> [Note] {
    let predicate = #Predicate<Note> { $0.title.localizedStandardContains(query) || $0.body.localizedStandardContains(query) }
    return try context.fetch(FetchDescriptor(predicate: predicate, sortBy: [SortDescriptor(\.createdAt, order: .reverse)]))
  }
  func add(_ note: Note) throws { context.insert(note); try context.save() }
  func delete(_ note: Note) throws { context.delete(note); try context.save() }
}
```

## 5. Auth manager (Keychain + refresh)

```swift
actor AuthManager {
  static let shared = AuthManager()
  private let keychain = Keychain(service: "com.example.app.auth")
  private var accessToken: String?
  private var refreshToken: String?
  private var expiryDate: Date?

  var token: String? {
    if let expiryDate, Date() > expiryDate { return nil }
    return accessToken
  }

  func login(email: String, password: String) async throws {
    let response = try await APIClient.shared.post("/auth/login", body: LoginRequest(email: email, password: password))
    try saveTokens(access: response.accessToken, refresh: response.refreshToken, expiresIn: response.expiresIn)
  }

  func refreshIfNeeded() async throws {
    guard let refreshToken, token == nil else { return }
    let response = try await APIClient.shared.post("/auth/refresh", body: RefreshRequest(refreshToken: refreshToken))
    try saveTokens(access: response.accessToken, refresh: response.refreshToken, expiresIn: response.expiresIn)
  }

  func logout() {
    accessToken = nil; refreshToken = nil; expiryDate = nil
    try? keychain.remove("accessToken"); try? keychain.remove("refreshToken")
  }

  private func saveTokens(access: String, refresh: String, expiresIn: Int) throws {
    accessToken = access; refreshToken = refresh
    expiryDate = Date().addingTimeInterval(TimeInterval(expiresIn))
    try keychain.set(access, key: "accessToken")
    try keychain.set(refresh, key: "refreshToken")
  }

  init() {
    accessToken = try? keychain.get("accessToken")
    refreshToken = try? keychain.get("refreshToken")
    // expiryDate not persisted — force refresh on next launch
  }
}
```

## 6. Image loading (Nuke/Kingfisher decision)

```swift
// SwiftUI AsyncImage (simple)
AsyncImage(url: url) { phase in
  switch phase {
  case .success(let image): image.resizable().scaledToFit()
  case .failure: Image(systemName: "exclamationmark.triangle")
  case .empty: ProgressView()
  @unknown default: EmptyView()
  }
}

// Kingfisher (caching + processors)
KFImage(url)
  .placeholder { ProgressView() }
  .resizable()
  .scaledToFill()
  .clipped()

// Nuke (modern, fast)
LazyImage(url: url) { state in
  if let image = state.image { image.resizable().scaledToFit() }
  else if state.error != nil { Image(systemName: "exclamationmark.triangle") }
  else { ProgressView() }
}
```

**Decision**: AsyncImage for simple cases; Kingfisher/Nuke for caching, placeholders, processors, prefetching.

## 7. Push notification setup

```swift
// AppDelegate
func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
  UNUserNotificationCenter.current().delegate = self
  application.registerForRemoteNotifications()
  return true
}

func application(_ application: UIApplication, didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data) {
  let token = deviceToken.map { String(format: "%02.2hhx", $0) }.joined()
  // Send token to server
  Task { try? await APIClient.shared.post("/devices", body: DeviceTokenRequest(token: token)) }
}

// UNUserNotificationCenterDelegate
extension AppDelegate: UNUserNotificationCenterDelegate {
  func userNotificationCenter(_ center: UNUserNotificationCenter,
    willPresent notification: UNNotification,
    withCompletionHandler completionHandler: @escaping (UNNotificationPresentationOptions) -> Void) {
    completionHandler([.banner, .list, .sound, .badge])
  }
  func userNotificationCenter(_ center: UNUserNotificationCenter,
    didReceive response: UNNotificationResponse,
    withCompletionHandler completionHandler: @escaping () -> Void) {
    let userInfo = response.notification.request.content.userInfo
    // Handle tap/action
    completionHandler()
  }
}
```

## 8. Widget extension skeleton

```swift
// Widget.swift
import WidgetKit
import SwiftUI

struct MyWidget: Widget {
  var body: some WidgetConfiguration {
    StaticConfiguration(kind: "com.example.widget", provider: Provider()) { entry in
      WidgetEntryView(entry: entry)
        .containerBackground(.fill.tertiary, for: .widget)
    }
    .configurationDisplayName("My Widget")
    .description("Shows latest data")
    .supportedFamilies([.systemSmall, .systemMedium, .systemLarge])
  }
}

struct Provider: TimelineProvider {
  func placeholder(in context: Context) -> Entry { Entry(date: .now, value: 42) }
  func getSnapshot(in context: Context, completion: @escaping (Entry) -> Void) {
    completion(Entry(date: .now, value: 42))
  }
  func getTimeline(in context: Context, completion: @escaping (Timeline<Entry>) -> Void) {
    Task {
      let value = try? await APIClient.shared.get("/widget-data")
      let entry = Entry(date: .now, value: value ?? 0)
      let timeline = Timeline(entries: [entry], policy: .atEnd)
      completion(timeline)
    }
  }
}

struct Entry: TimelineEntry { let date: Date; let value: Int }

struct WidgetEntryView: View {
  var entry: Provider.Entry
  var body: some View {
    VStack { Text("\(entry.value)").font(.title); Text("Latest").font(.caption) }
  }
}

@main struct Widgets: WidgetBundle {
  var body: some Widget { MyWidget() }
}
```

## 9. Live Activity (ActivityKit)

```swift
struct WorkoutAttributes: ActivityAttributes {
  struct ContentState: Codable, Hashable {
    var progress: Double
    var heartRate: Int
  }
  var workoutType: String
}

// Start
let attributes = WorkoutAttributes(workoutType: "Running")
let state = WorkoutAttributes.ContentState(progress: 0.0, heartRate: 0)
let activity = try Activity.request(attributes: attributes, contentState: state, pushType: .token)

// Update
await activity.update(ActivityContent(state: WorkoutAttributes.ContentState(progress: 0.5, heartRate: 150), staleDate: nil))

// End
await activity.end(ActivityContent(state: WorkoutAttributes.ContentState(progress: 1.0, heartRate: 0), staleDate: nil), dismissalPolicy: .default)

// Widget UI
struct WorkoutLiveActivity: Widget {
  var body: some WidgetConfiguration {
    ActivityConfiguration(for: WorkoutAttributes.self) { context in
      // Lock screen UI
      VStack { Text(context.attributes.workoutType); ProgressView(value: context.state.progress) }
    } dynamicIsland: { context in
      DynamicIsland {
        DynamicIslandExpandedRegion(.leading) { Text("Run") }
        DynamicIslandExpandedRegion(.trailing) { Text("\(context.state.heartRate) BPM") }
        DynamicIslandExpandedRegion(.bottom) { ProgressView(value: context.state.progress) }
      } compactLeading: { Image(systemName: "figure.run") }
      compactTrailing: { Text("\(Int(context.state.progress * 100))%") }
      minimal: { Image(systemName: "figure.run") }
    }
  }
}
```

## 10. App Intents (Siri AI integration)

```swift
struct OrderItemIntent: AppIntent {
  static var title: LocalizedStringResource = "Order Item"
  @Parameter(title: "Item") var item: ItemEntity
  @Parameter(title: "Quantity") var quantity: Int = 1
  static var parameterSummary: some ParameterSummary {
    Summary("Order \(\.$quantity) \(\.$item)")
  }
  func perform() async throws -> some IntentResult & ProvidesDialog {
    try await OrderService.shared.placeOrder(item: item, quantity: quantity)
    return .result(dialog: "Ordered \(quantity) \(item.name)")
  }
}

struct ItemEntity: AppEntity {
  var id: String
  var name: String
  static var typeDisplayRepresentation = TypeDisplayRepresentation(name: "Item")
  var displayRepresentation: DisplayRepresentation { DisplayRepresentation(title: "\(name)") }
  static var defaultQuery = ItemQuery()
}

struct ItemQuery: EntityQuery {
  func entities(for identifiers: [String]) async throws -> [ItemEntity] {
    try await OrderService.shared.items(ids: identifiers)
  }
  func suggestedEntities() async throws -> [ItemEntity] {
    try await OrderService.shared.suggestedItems()
  }
}

struct AppShortcuts: AppShortcutsProvider {
  static var appShortcuts: [AppShortcut] {
    AppShortcut(intent: OrderItemIntent(), phrases: ["Order \(.applicationName)"],
      shortTitle: "Order", systemImageName: "cart")
  }
}
```

## 11. Pre-ship checklist

### Code quality
- [ ] No force unwraps/`try!`/`as!` in production code.
- [ ] No `print` statements (use `Logger`/`os_log`).
- [ ] No hardcoded secrets/keys.
- [ ] Error handling on all throwing calls.
- [ ] Memory leaks checked (Instruments).
- [ ] Concurrency warnings fixed (Swift 6 strict).
- [ ] Accessibility audit passed.

### Privacy & compliance
- [ ] Privacy manifest (`PrivacyInfo.xcprivacy`) included.
- [ ] Usage descriptions for all permissions.
- [ ] ATT prompt implemented (if tracking).
- [ ] App Store nutrition labels accurate.
- [ ] Third-party SDK privacy manifests included.
- [ ] Data collection declared correctly.

### Performance
- [ ] Launch time < 2s (cold).
- [ ] No main-thread blocking.
- [ ] Images downsampled/cached.
- [ ] Lists use lazy loading.
- [ ] Instruments pass (Time Profiler, Allocations, Leaks).
- [ ] Battery impact minimal.

### UI/UX
- [ ] Dark mode tested.
- [ ] Dynamic Type tested.
- [ ] RTL layout tested.
- [ ] All device sizes tested (iPhone, iPad, Mac, Watch, TV, Vision).
- [ ] Empty states, loading states, error states designed.
- [ ] Offline behavior handled.
- [ ] Accessibility labels/hints added.
- [ ] Haptic feedback where appropriate.

### Distribution
- [ ] Version/build numbers correct.
- [ ] Entitlements match capabilities.
- [ ] Signing configured (automatic or manual).
- [ ] Archive succeeds.
- [ ] `altool --validate-app` passes.
- [ ] Export compliance declared.
- [ ] Screenshots for all required sizes.
- [ ] Metadata complete.
- [ ] Privacy policy URL valid.
- [ ] Age rating accurate.

## 12. Decision cheat sheets

### Persistence
| Need | Use |
|---|---|
| Settings/flags | `UserDefaults`/`@AppStorage` |
| Secrets | Keychain |
| Structured data | SwiftData |
| Complex queries | GRDB.swift |
| Cloud sync | CloudKit/`CKSyncEngine` |
| Documents | Files + security-scoped bookmarks |

### Networking
| Need | Use |
|---|---|
| Simple REST | `URLSession` + Codable |
| Complex auth/retry | Alamofire |
| GraphQL | Apollo iOS |
| WebSocket | `URLSessionWebSocketTask` |
| Low-level TCP/UDP | `NWConnection`/`NWListener` |
| Background transfer | `URLSessionConfiguration.background` |

### UI
| Need | Use |
|---|---|
| New app UI | SwiftUI |
| Complex custom UI | UIKit/AppKit |
| Long list | `List`/`LazyVStack` |
| Master-detail | `NavigationSplitView` |
| Modal | `.sheet`/`.fullScreenCover` |
| Web content | `WKWebView` |
| Camera | AVFoundation `AVCaptureSession` |
| Maps | MapKit `Map`/`MKMapView` |
| Charts | Swift Charts |

### Architecture
| Need | Use |
|---|---|
| Simple app | SwiftUI + `@Observable` + SwiftData |
| Complex app | TCA or MVVM + repositories |
| DI | Environment/`@Observable` dependencies |
| Testing | Swift Testing + XCTest |
| Project gen | XcodeGen/Tuist |
| Packages | SwiftPM |
