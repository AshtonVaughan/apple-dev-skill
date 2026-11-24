# Swift Concurrency, Combine & Observation

Swift 6 strict concurrency + Combine + the Observation framework. Read `01-swift-language.md` for language basics.

## 1. The concurrency model in one paragraph

Swift concurrency = **cooperative** threads managed by the runtime, not OS threads per task. `async` functions run on the cooperative thread pool; `await` marks suspension points where the thread is released. Isolation domains (actors, `@MainActor`) guarantee exclusive access to mutable state; `Sendable` types may cross boundaries. The compiler enforces this statically in Swift 6 mode.

## 2. async/await basics

```swift
func fetchUser() async throws -> User { ... }
let user = try await fetchUser()

// async properties/methods on types
class Repo { func load() async -> [Item] { ... } }

// async sequences
for await line in URLSession.shared.bytes(from: url).lines { ... }
let values = await [1,2,3].asyncMap { await process($0) }  // CollectionConcurrencyKit / swift-async-algorithms
```
- `async` = may suspend; `await` = potential suspension point (NOT necessarily thread switch).
- `async let` for parallel structured subtasks:
  ```swift
  async let a = fetchA(); async let b = fetchB()
  let (x, y) = try await (a, b)   // implicit await at use; both must finish or scope cancels
  ```
- `Task.sleep(for:)` / `Task.sleep(nanoseconds:)`; `ContinuousClock`, `SuspendingClock` for testable time.
- `withTaskGroup` / `withThrowingTaskGroup` for dynamic parallelism:
  ```swift
  try await withThrowingTaskGroup(of: Item.self) { group in
    for id in ids { group.addTask { try await fetch(id) } }
    for try await item in group { results.append(item) }  // completion order
  }
  ```
  Group waits for all children; first throw cancels rest. `group.addTaskUnlessCancelled`, `next()`, `waitForAll()`.
- `withDiscardingTaskGroup` — fire-and-forget children that don't produce values.

## 3. Tasks

| API | Lifetime | Cancellation | Use |
|---|---|---|---|
| `async let` | structured, scope-bound | auto on scope exit | parallel subtasks |
| `Task {}` | inherits actor+priority, unstructured | manual or parent-cancel-aware | start async work from sync context (view `.task`) |
| `Task.detached {}` | no inherited actor/priority | manual | truly independent background work |
| `Task.immediate {}` | Swift 6.x: runs synchronously until first suspension | manual | latency-critical start |
| `Task(priority:)` | QoS hint: `.userInitiated`, `.utility`, `.background`, `.high` | — | |

- `Task.isCancelled` / `Task.checkCancellation()` (throws `CancellationError`); `withTaskCancellationHandler(operation:onCancel:)` for bridging cancellable APIs (e.g. URLSession task cancel).
- **Swift 6.4**: `withTaskCancellationShield { ... }` — temporarily prevents cancellation delivery inside the region (for critical cleanup/commit phases).
- `Task.local` values via `@TaskLocal` — propagate context (trace IDs, user IDs) through awaits without parameter threading.
- Unstructured tasks are NOT auto-cancelled — store handle if you need to cancel; leaked tasks still get cancelled when app backgrounds only if system decides (not guaranteed). `.task {}` view modifier auto-cancels on disappear.
- Priority inversion: UI waits on `.background` work → bump priority or restructure; check Instruments Swift Concurrency template.

## 4. Actors

```swift
actor Counter { var value = 0; func increment() { value += 1 } }
let c = Counter()
await c.increment()          // cross-actor call requires await
task { await c.value }       // isolated property access needs await too
```
- `actor` = reference type with isolated mutable state; all members isolated by default.
- `nonisolated let/var` — immutable lets and `nonisolated` computed props accessible without await (safe because immutable).
- `nonisolated(unsafe)` — escape hatch for "I know it's safe" (e.g. immutable reference types, OS handles). Audit carefully.
- **Reentrancy**: an actor method suspends at `await` — another task may interleave and mutate state. Pattern: re-fetch state after every `await`, or use `isolated` synchronous helpers for atomic check-and-set.
- `isolated` parameters (SE-0313/Swift 5.5+): `func f(_ other: isolated Counter)` — synchronous access to another actor's state when caller already isolated.
- Global actors: `@MainActor`, `@UIApplicationMainActor`? (no — just `@MainActor`), custom `@globalActor` (e.g. `@DatabaseActor`). `@MainActor` on a class isolates all members; `nonisolated` escapes.
- `MainActor.assumeIsolated { }` — synchronous bridge when runtime guarantees main thread (precondition failure if wrong).
- `MainActor.run { }` — hop to main from anywhere.
- Distributed actors (`distributed actor`, `DistributedActorSystem`) — experimental; for actor clusters across processes.

## 5. Sendable & isolation

```swift
struct User: Sendable { let id: Int; let name: String }   // all-Sendable members → inferred Sendable (internal)
final class Config: @unchecked Sendable { /* immutable after init, thread-safe internals */ }
enum Phase: Sendable { case idle, loading, loaded }
```
- Value types infer `Sendable` when all members are Sendable (internal types only — public needs explicit conformance).
- Classes NEVER infer Sendable; mark `final` + immutable `let` Sendable members → can conform; mutable shared state → `@unchecked Sendable` ONLY with proven thread safety (locks, atomics, immutability).
- `sending` keyword (Swift 6): `func f(_ x: sending MyType)` / `-> sending MyType` — ownership transfers, disconnecting from old isolation domain. Prevents shared references from crossing.
- Region-based isolation: compiler analyzes access regions — a `let` constant's region can be sent even from non-Sendable types in some cases.
- `@preconcurrency import Foo` — treat module's types as Sendable-ish where safe; for legacy deps.
- Common fixes: make DTOs `Sendable` structs; mark closures `@Sendable` when crossing actors; `nonisolated` for pure functions on actor types; move shared state into an `actor` or protect with `Mutex` (Synchronization framework).
- **Synchronization framework** (Swift 6.2+, `import Synchronization`): `Mutex<Value>` (lock-guarded value, `withLock`), `Atomic` types (AtomicInt etc.), `AtomicReference`, `ManagedAtomic` — for lock-free/shared-state needs inside `@unchecked Sendable` types. Prefer over NSLock/os_unfair_lock in new code.

## 6. Bridging completion handlers → async

```swift
// Checked continuation — crashes on double-resume or leak (debugging aid)
let data = try await withCheckedThrowingContinuation { cont in
  URLSession.shared.dataTask(with: url) { data, _, error in
    if let error { cont.resume(throwing: error) }
    else { cont.resume(returning: data!) }
  }.resume()
}

// Unsafe variants — YOU must guarantee exactly-once resume
withUnsafeContinuation { ... }

// AsyncStream for delegate/callback streams
let (stream, continuation) = AsyncStream<Int>.makeStream()
continuation.yield(1); continuation.finish()
// AsyncThrowingStream for failable streams; .bufferingNewest(n) policy for backpressure
```
- Checked continuations trap on programmer error (leak = continuation deallocated without resume → runtime warning + leak; double resume = crash). Use checked in app code; unsafe only in perf-critical wrappers where invariant is proven.
- Delegate-based APIs (URLSessionDownloadDelegate, AVFoundation, Vision) → wrap in `AsyncStream`/`AsyncThrowingStream` or continuation. Pattern: create stream, hold continuation in delegate, `finish()` in `didComplete`/`deinit`.
- Combine → async: `publisher.values` (AsyncPublisher, iOS 15+), `publisher.first()`, sink→continuation.

## 7. Combine (still relevant — interop, timers, UIKit publishers)

```swift
import Combine
cancellable = publisher
  .debounce(for: .milliseconds(300), scheduler: RunLoop.main)
  .removeDuplicates()
  .receive(on: DispatchQueue.main)
  .sink { value in ... } receiveCompletion: { ... }
  .store(in: &cancellables)   // Set<AnyCancellable>
```
- Core types: `AnyPublisher`, `PassthroughSubject`, `CurrentValueSubject`, `@Published` (inside ObservableObject), operators (`map`, `compactMap`, `flatMap` latest vs merge, `switchToLatest`, `combineLatest`, `zip`, `merge`, `throttle`, `debounce`, `scan`, `reduce`, `prepend`, `append`, `drop`, `prefix`, `removeDuplicates`, `eraseToAnyPublisher`).
- Schedulers: `DispatchQueue.main`, `RunLoop.main`, `OperationQueue`, `DispatchQueue.global()`.
- Backpressure: Combine has NO backpressure — fast publishers flood slow subscribers. Mitigate: `throttle`, `debounce`, buffer in subject, or move to `AsyncStream` with `.buffering*` policy.
- When to use vs async sequences: Combine for UIKit/Combine-heavy legacy code, Timers publishers (`Timer.publish`), NotificationCenter (`NotificationCenter.default.publisher`), URLSession dataTaskPublisher, CoreData publisher; async/await + AsyncSequence for new code.
- SwiftUI's old data flow (`ObservableObject` + `@Published` + `objectWillChange`) is Combine under the hood — still works, but `@Observable` is the modern replacement.

## 8. Observation framework (modern SwiftUI state)

```swift
@Observable
final class Store {
  var items: [Item] = []
  var filter = "" { didSet { /* Observation tracks reads automatically */ } }
}
// In view:
@State private var store = Store()         // or @Bindable for child editing
@Environment(Store.self) private var store  // shared via .environment(store)
@Query private var items: [Item]            // SwiftData auto-tracking
```
- `@Observable` macro instruments stored properties — views re-render only when a *read* property changes (fine-grained tracking, no `objectWillChange` storms).
- `withObservationTracking { read } onChange: { write }` — imperative tracking for non-view code.
- `@Observable` works on classes only; `@State` still needed for value ownership in views; `@Bindable` for passing editable bindings to children; `@Environment(Store.self)` for DI.
- Migration: `ObservableObject` + `@Published` still fully supported; `@Observable` doesn't support `willSet`-dependent logic the same way — test carefully when migrating.

## 9. Common pitfalls & fixes

| Symptom | Cause | Fix |
|---|---|---|
| "Main actor-isolated class property cannot be used" | touching `@MainActor` state synchronously | `await MainActor.run`, mark caller `@MainActor`, or `nonisolated` pure parts |
| Actor method slow | reentrancy forcing serialization? actually actors don't serialize across awaits — check lock contention / thread pool starvation | profile; reduce awaits inside actor; batch work |
| `Task {}` leaks/crashes on cancel | capturing non-Sendable self or not checking cancellation | capture `[weak self]`, `try Task.checkCancellation()`, store handle |
| UI freezes despite `await` | `await` on nonisolated sync work that blocks thread (sync I/O, `DispatchSemaphore.wait`, `RunLoop.main.run`) | move blocking work to `Task.detached` or make API async; never block cooperative pool threads |
| Deadlock: `await` inside `DispatchQueue.main.sync` on main | blocking main waiting for main | restructure — use continuation or async all the way |
| Data race despite actors | `@unchecked Sendable` lie, `nonisolated(unsafe)` misuse, or C callback mutating state | audit shared refs; use `Mutex`/`Atomic`; verify with TSan |
| Race: two tasks read-modify-write actor counter | each `await` boundary interleaves | make mutation a single actor-isolated sync method (no awaits inside) |
| Combine sink stops firing | missing `.store(in:)` → cancelled immediately | always store in `cancellables` owned by long-lived object |
| `@Published` doesn't fire | publishing during init / equal values (no `removeDuplicates` needed but `@Published` skips in willSet? — actually fires always) | check subscription timing; use `CurrentValueSubject` if need current value |
| Memory: continuation leaked | never resumed on all paths | `defer { cont.resume(...) }` pattern or checked continuation warnings |

## 10. Debugging concurrency

- **Strict concurrency warnings**: build with `SWIFT_STRICT_CONCURRENCY = complete` (or `-strict-concurrency=complete`) to surface issues before Swift 6 mode.
- **Thread Sanitizer**: Product > Scheme > Edit > Run > Diagnostics > Thread Sanitizer — catches actual races at runtime.
- **Instruments → Swift Concurrency template**: shows tasks, priority inversions, executor issues.
- Runtime warnings: "SWIFT TASK CONTINUATION MISUSE", "instance of X was deallocated before continuation resumed" — grep console.
- `Task.name = "FetchUser"` — names appear in debugger/Instruments.
- `os_signpost` intervals around suspected slow sections.
- `dispatchPrecondition(condition: .onQueue(.main))` for debugging main-thread assumptions.

## 11. Modern patterns

```swift
// Repository pattern with actor
actor ItemRepository {
  private var cache: [UUID: Item] = [:]
  func item(id: UUID) async throws -> Item {
    if let cached = cache[id] { return cached }
    let item = try await api.fetchItem(id: id)
    cache[id] = item
    return item
  }
}

// MainActor view model calling actor repo
@MainActor @Observable
final class ItemListModel {
  private let repo: ItemRepository
  var items: [Item] = []
  var isLoading = false
  init(repo: ItemRepository) { self.repo = repo }
  func load() async {
    isLoading = true; defer { isLoading = false }
    items = (try? await repo.allItems()) ?? []
  }
}

// AsyncSequence bridge for delegates
func locationUpdates() -> AsyncStream<CLLocation> {
  AsyncStream { continuation in
    let manager = CLLocationManager()
    manager.delegate = /* object that calls continuation.yield */
    continuation.onTermination = { _ in manager.stopUpdatingLocation() }
  }
}
```
