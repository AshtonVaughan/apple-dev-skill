# Advanced Concurrency — Internals & Expert Patterns

Beyond `02-concurrency.md`: custom executors, low-level synchronization, distributed actors, isolation internals, testing concurrency.

## 1. Custom executors

```swift
import Dispatch

// Custom serial executor
final class SerialExecutor: SerialExecutor, @unchecked Sendable {
  private let queue = DispatchQueue(label: "com.example.serial")
  func enqueue(_ job: UnownedJob) {
    queue.async { job.runSynchronously(on: self.asUnownedSerialExecutor()) }
  }
  func asUnownedSerialExecutor() -> UnownedSerialExecutor {
    UnownedSerialExecutor(ordinary: self)
  }
}

// Custom actor executor
actor MyActor {
  nonisolated var unownedExecutor: UnownedSerialExecutor {
    UnownedSerialExecutor(ordinary: SerialExecutor())
  }
}

// Custom global actor
@globalActor
actor DatabaseActor {
  static let shared = DatabaseActor()
  private let executor = SerialExecutor()
  nonisolated var unownedExecutor: UnownedSerialExecutor {
    UnownedSerialExecutor(ordinary: executor)
  }
}
```

- `SerialExecutor` protocol — `enqueue(_:)` for jobs; `asUnownedSerialExecutor()`.
- `UnownedSerialExecutor` — `ordinary:` init for custom executors.
- `Job`/`UnownedJob` — `runSynchronously(on:)`.
- Custom executors must guarantee: jobs run on the executor's thread/context; no concurrent execution (serial) or documented concurrency.
- `DispatchQueue` as executor: `queue.asUnownedSerialExecutor()`? — `DispatchQueue` conforms to `SerialExecutor`? — `DispatchQueue` has `asUnownedSerialExecutor()`? — check: `DispatchQueue` conforms to `SerialExecutor` via `enqueue`.
- `Task` executor: `Task(executorPreference:)`? — `Task(priority:operation:)` inherits; custom executor via `@_inheritActorContext`? — `Task` doesn't take executor directly; use `actor`/`@globalActor` or `withTaskExecutorPreference`? — check current API.

## 2. Low-level synchronization

### Synchronization framework (Swift 6.2+)
```swift
import Synchronization

// Mutex — lock-guarded value
let counter = Mutex(0)
counter.withLock { $0 += 1 }
counter.withLock { value in
  // value is inout — mutate directly
}

// Atomic types
let atomicInt = Atomic<Int>(0)
atomicInt.add(1, ordering: .relaxed)
let value = atomicInt.load(ordering: .acquiring)
atomicInt.store(42, ordering: .releasing)
atomicInt.compareExchange(expected: 0, desired: 1, ordering: .acqRel)
atomicInt.exchange(5, ordering: .seqCst)

// AtomicReference for objects
let atomicRef = AtomicReference<MyClass?>(nil)
atomicRef.store(newObj, ordering: .release)
let obj = atomicRef.load(ordering: .acquire)

// ManagedAtomic (pointer-based)
let managed = ManagedAtomic<Int>(0)
managed.wrappingAdd(1, ordering: .relaxed)
```

- `Mutex<Value>` — `withLock(_:)`, `withLockUnchecked(_:)`; `State` type for recursive? — `Mutex` is non-recursive.
- `Atomic<Value>` — `load`, `store`, `exchange`, `compareExchange`, `add`, `sub`, `and`, `or`, `xor`, `fetchAnd`, `fetchOr`, `fetchXor`, `fetchAdd`, `fetchSub`, `fetchAnd`, `fetchOr`, `fetchXor` — for integer types.
- `AtomicReference<Value>` — `load`, `store`, `exchange`, `compareExchange` for class references.
- `ManagedAtomic<Value>` — same as Atomic but for stack-allocated values.
- `AtomicOptional`? — `Atomic<Optional<T>>`.
- Ordering: `.relaxed`, `.acquiring`, `.releasing`, `.acqRel`, `.seqCst`.
- `Mutex` `withLock` — returns value; `withLockUnchecked` for non-Sendable? — check.
- `OSAllocatedUnfairLock` (iOS 16+, macOS 13+) — `os_unfair_lock` wrapper: `lock()`, `unlock()`, `withLock(_:)`, `state`.
- `NSLock`/`NSRecursiveLock` — Foundation locks; `lock()`, `unlock()`, `try()`, `lock(before:)`.
- `NSCondition`/`NSConditionLock` — condition variables.
- `DispatchSemaphore` — `wait()`, `signal()`, `wait(timeout:)`.
- `pthread_mutex`/`pthread_rwlock` — C-level locks via `import Darwin`/`import Glibc`.

### Actor reentrancy patterns
```swift
actor BankAccount {
  private var balance: Int = 0

  // BAD: reentrancy can change balance between await and mutation
  func transfer(to other: BankAccount, amount: Int) async throws {
    if balance >= amount {           // check
      await other.deposit(amount)    // SUSPENDS — another task can withdraw!
      balance -= amount              // now balance might be < amount
    }
  }

  // GOOD: atomic check-and-set (no await between check and mutation)
  func transferAtomic(to other: BankAccount, amount: Int) async throws {
    try await withCheckedThrowingContinuation { cont in
      Task { await self.performTransfer(to: other, amount: amount, cont: cont) }
    }
  }

  private func performTransfer(to other: BankAccount, amount: Int,
    cont: CheckedContinuation<Void, Error>) async {
    // This runs on actor — no suspension between check and mutation
    guard balance >= amount else { cont.resume(throwing: InsufficientFunds()); return }
    balance -= amount
    await other.deposit(amount)      // deposit may suspend, but balance already updated
    cont.resume()
  }
}
```

- **Reentrancy**: actor methods suspend at `await` — other tasks can interleave. Pattern: keep check-and-mutate atomic (no `await` between), or re-validate after `await`.
- `isolated` synchronous helpers: `private func` on actor is isolated — call from actor methods without `await`.
- `nonisolated` for pure functions/stateless helpers on actor types.
- `nonisolated(unsafe)` for "I know it's safe" — audit carefully.
- `Task` inheritance: `Task {}` inherits actor context + priority; `Task.detached` doesn't.
- `MainActor.assumeIsolated` — sync bridge when runtime guarantees main thread.
- `MainActor.run` — hop to main from anywhere.

## 3. Distributed actors & advanced isolation

```swift
// Distributed actor (experimental)
distributed actor Player {
  distributed let id: UUID
  var score: Int = 0

  distributed func increment() { score += 1 }
}

// Isolated conformances
extension Foo: @MainActor SomeProtocol { }
// or
extension Foo: SomeProtocol where Self: @MainActor { }

// Isolated parameters
func process(_ data: isolated Data) { /* synchronous access to isolated data */ }
```

- `distributed actor` — actor across processes; `DistributedActorSystem` for cluster.
- `isolated` parameters — synchronous access to another actor's state when caller already isolated.
- `isolated` conformances — protocol conformance isolated to specific actor.
- `sending` parameters/results — ownership transfer across isolation.
- `nonisolated(nonsending)` — nonisolated but doesn't send (Swift 6.2+).
- `@_inheritActorContext` — inherit actor context (internal).
- `Task.immediate` — run synchronously until first suspension (Swift 6.x).
- `withTaskGroup`/`withThrowingTaskGroup` — structured concurrency.
- `withDiscardingTaskGroup` — fire-and-forget children.
- `async let` — parallel structured subtasks.
- `TaskGroup.addTask`/`addTaskUnlessCancelled` — add child tasks.
- `TaskGroup.next()` — get next completed result.
- `TaskGroup.waitForAll()` — wait for all children.
- `TaskLocal` — `@TaskLocal` for context propagation.
- `withTaskCancellationHandler` — cancellation handler.
- `withTaskCancellationShield` — shield from cancellation (Swift 6.4).
- `withCheckedContinuation`/`withUnsafeContinuation`/`withCheckedThrowingContinuation`/`withUnsafeThrowingContinuation` — bridge to async.
- `CheckedContinuation`/`UnsafeContinuation` — `resume(returning:)`, `resume(throwing:)`.
- `AsyncStream`/`AsyncThrowingStream` — `makeStream()`, `yield`, `finish`, `bufferingPolicy` (`.unbounded`, `.bufferingOldest`, `.bufferingNewest`).
- `AsyncSequence` — `map`, `compactMap`, `filter`, `prefix`, `dropFirst`, `prefix(while:)`, `drop(while:)`, `joined`, `flatMap`, `merge`, `zip`, `combineLatest`, `debounce`, `throttle`, `timer`? — `AsyncAlgorithms` package.
- `AsyncAlgorithms` (swift-async-algorithms) — `AsyncTimerSequence`, `AsyncDebounceSequence`, `AsyncChannel`, `AsyncThrowingChannel`, `AsyncMergeSequence`, `AsyncZipSequence`, `AsyncCombineLatestSequence`, `AsyncThrowingMapSequence`? — check.

## 4. Testing concurrency

```swift
import Testing

@Test func actorIsolation() async {
  let counter = Counter()
  await withTaskGroup(of: Void.self) { group in
    for _ in 0..<100 {
      group.addTask { await counter.increment() }
    }
  }
  #expect(await counter.value == 100)
}

@Test func cancellation() async throws {
  let task = Task {
    try await Task.sleep(for: .seconds(10))
    return "done"
  }
  task.cancel()
  #expect(throws: CancellationError.self) {
    try await task.value
  }
}

@Test func sendableCheck() {
  // Compile-time check — won't compile if non-Sendable crosses boundary
  let user = User(name: "Test")  // struct: Sendable
  Task { print(user.name) }
}
```

- `withTaskGroup` for parallel test execution.
- `Task.cancel()` + `Task.checkCancellation()` for cancellation tests.
- `withCheckedContinuation` for async test bridges.
- `XCTestExpectation` for callback-based async.
- `Confirmation` (Swift Testing) for async callbacks.
- `@MainActor` tests — `@MainActor` on test class/func.
- `nonisolated` tests — escape actor isolation.
- `Task.sleep` — deterministic delays; `ContinuousClock`/`SuspendingClock` for testable time.
- `Mutex`/`Atomic` for thread-safe test state.
- `OSAllocatedUnfairLock` for lock-based test coordination.
- `NSLock`/`DispatchSemaphore` for test barriers.
- `Task.yield()` — yield to other tasks.
- `Task.detached` — test truly concurrent behavior.
- `Task(priority:)` — test priority ordering.
- `TaskLocal` — test context propagation.
- `withTaskGroup` — test structured concurrency.
- `withThrowingTaskGroup` — test error handling.
- `async let` — test parallel execution.
- `MainActor.run` — test main actor hops.
- `assumeIsolated` — test main thread assumptions.
- `nonisolated` — test non-isolated access.
- `sending` — test ownership transfer.
- `consuming`/`borrowing` — test ownership semantics.

## 5. Common advanced concurrency bugs → fixes

| Bug | Cause | Fix |
|---|---|---|
| Actor reentrancy race | await between check and mutate | keep atomic, re-validate after await |
| Task leak | unstructured task never cancelled | store handle, cancel on disappear |
| Priority inversion | UI waits on background work | bump priority, restructure |
| Sendable warning ignored | `@unchecked Sendable` lie | audit, use Mutex/Atomic, prove safety |
| Continuation leak | never resumed | `defer { cont.resume(...) }` |
| AsyncSequence buffer overflow | fast producer, slow consumer | `.bufferingNewest(n)` policy |
| Deadlock on main | `DispatchQueue.main.sync` on main | use continuation, never block main |
| Combine backpressure | no backpressure in Combine | throttle/debounce, move to AsyncStream |
| Task cancellation ignored | not checking `isCancelled` | `Task.checkCancellation()`, `withTaskCancellationHandler` |
| Actor hop overhead | excessive actor calls | batch operations, reduce awaits |
| `@MainActor` assumption wrong | calling from background | check `Thread.isMainThread`, use `MainActor.run` |
| Distributed actor slow | network latency | use for cross-device, not local |
| `Mutex` deadlock | recursive lock attempt | use `NSRecursiveLock` or restructure |
| `Atomic` ordering wrong | wrong memory ordering | use `.seqCst` for strict, `.relaxed` for perf |
| `TaskLocal` not propagating | not set before task creation | set `TaskLocal` value before `Task {}` |
| `AsyncStream` finish not called | continuation leaked | call `finish()` in `onTermination`/`deinit` |
| `withTaskGroup` child hang | child never completes | ensure child returns/throws |
| `Task.detached` loses context | no inherited actor/priority | use `Task {}` for inherited context |
| `MainActor.assumeIsolated` crash | called off main thread | verify main thread first |
| `nonisolated(unsafe)` data race | unsafe escape hatch | audit, use Mutex/Atomic instead |
