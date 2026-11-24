# Swift Language Reference (Swift 6.4.2)

Complete language reference for modern Swift. Assumes Swift 6 language mode (strict concurrency) unless noted.

## 1. Type system fundamentals

### Value vs reference types
- **Structs/enums**: value semantics — copied on assignment (with copy-on-write optimization for Array/Dictionary/String/Set). Mutating a copy never affects the original.
- **Classes/actors**: reference semantics — assignment shares identity. `===`/`!==` compare identity; `==` needs `Equatable`.
- Prefer structs for data; classes only when you need identity, inheritance, or deinit side effects. Actors for shared mutable state (see `02-concurrency.md`).

### Optionals
```swift
var name: String? = nil
if let name { ... }              // shorthand unwrap (Swift 5.7+)
guard let name else { return }   // early exit
guard let name = name else { ... }
let count = items?.count ?? 0    // nil-coalescing
items?.forEach { ... }           // optional chaining
// Swift 6.4 (SE-0521): no parens needed — `some P?` and `any P?` are valid
```
- `T!` (implicitly unwrapped) — avoid; use only for outlets set by framework and `lazy` patterns.
- `Optional` is an enum: `.some`/`.none`.

### Tuples, ranges, collections
```swift
let pair = (x: 1, y: 2); pair.x
let range = 1..<5; let closed = 1...5   // Range, ClosedRange
let dict = ["a": 1]; dict["a"]          // Dictionary subscript → Value?
let set: Set<Int> = [1, 2, 3]
```
- Dictionary/Set subscript returns `Value?` — missing key is `nil`, NOT an error.
- `Array` literal inference: `[1, 2, 3]` → `[Int]`; heterogeneous needs explicit type.
- Swift 6.4: arrays can hold **noncopyable elements** without COW overhead; `Iterable` protocol for iterating without copies (see `copy`/`consume` below).

### Strings & Unicode
- `String` is a collection of `Character` (grapheme clusters), NOT bytes. `"café".count` may differ from `utf8.count`, `utf16.count`, `unicodeScalars.count`. Never index strings by integer offsets — use `String.Index` (`startIndex`, `index(after:)`, `firstIndex(of:)`, ranges).
- `Substring` shares storage with parent `String` — cheap slicing; convert to `String` to persist.
- Interpolation: `"\(value)"` — supports `String(format:)`-style via `appendInterpolation`, and `LocalizedStringKey` interpolation for localized strings in SwiftUI `Text`.
- `LocalizedStringKey` ≠ `String`: `Text("Hello \(name)")` treats literal as localization key; `Text(verbatim:)` skips lookup.
- **Regex literals** (Swift 5.7+): `let re = /\d+/` or `#/.../#` for extended syntax; `Regex<Output>` with `(\d+)` capture → `Substring` matches; `wholeMatch`, `firstMatch`, `replacing`, `split`. `RegexBuilder` for multi-line composition. `#Predicate` macro builds regexes for SwiftData/Core Data.
- `String(format:)` needs Foundation; prefer interpolation. `String(localized:)` for explicit localization.

## 2. Functions & closures

```swift
func greet(_ name: String, age: Int = 0, _ extra: String... ) -> String  // variadic must be last-ish, array inside
func f(_ x: inout Int)                    // inout = mutable borrow, copy in/copy out
func throwing() throws -> Int             // typed throws: `throws(MyError)` in Swift 6
func rethrowing(_ f: () throws -> Void) rethrows
```
- Function types: `(Int, String) -> Bool`; throwing: `(Int) throws -> Bool`; async: `(Int) async -> Bool`; both: `(Int) async throws -> Bool`.
- `@escaping` closures outlive the call (stored in properties); `@autoclosure` wraps expression in closure for `&&`/`??`/`assert`-style laziness.
- Trailing closure syntax: `foo { ... }`, `foo(arg) { ... }`, multiple labeled: `foo { a in } label: { b in }` (Swift 5.3+). **Swift 6.4 (SE-0508)**: trailing closures after array literals now parse correctly — `foo([1,2]) { ... }` unambiguous.
- Closures capture by reference; capture list `[weak self]`, `[unowned self]`, `[x = expr]` for control. `self.` required explicitly when capturing self in escaping closures.
- `defer { ... }` runs at scope exit (all paths incl. throws). **Swift 6.4 (SE-0493)**: `defer` bodies may contain `await` — isolation matches enclosing scope.

## 3. Generics, protocols, opaque/existential types

```swift
func map<T, U>(_ xs: [T], _ f: (T) -> U) -> [U] where T: Equatable { ... }
protocol Container<Element> { associatedtype Element; mutating func add(_ e: Element) }
extension Container where Element == Int { func sum() -> Int { ... } }   // constrained extension
```
- **Primary associated types** (Swift 5.7): `protocol Collection<Element>` → `any Collection<Int>` spells the generic.
- **`some P`** = opaque result type (one concrete type, hidden) — for return values and parameter packs. **`any P`** = existential box (any conforming type, dynamic dispatch). Rule of thumb: functions *return* `some`, *accept* `any` unless generic needed.
- **Swift 6.4 (SE-0521)**: `some P?` / `any P?` — no parentheses required.
- **Parameter packs** (variadic generics, Swift 5.9): `each T`, `repeat each T`, pack iteration `for t in repeat each pack`.
- Protocol extensions provide default implementations; a type's own method always wins over extension default (no dynamic dispatch for defaults unless requirement).
- `~Copyable`/`~Escapable` on associated types (SE-0503, Swift 6.4): relax protocol requirements to allow noncopyable/non-escaping witnesses.
- `Sendable`, `Equatable`, `Hashable`, `Codable` (= Encodable & Decodable), `Comparable`, `CustomStringConvertible`, `Identifiable` (id property), `Observable` (Observation framework), `AppIntent`-family — the standard library of protocols.
- `CaseIterable` (allCases), `RawRepresentable`, `LosslessStringConvertible`, `ExpressibleBy*Literal` (why `"x"` works as `String`, `5` as `Double`, etc.).

## 4. Enums — the most powerful type

```swift
enum Result<T, E: Error> { case success(T); case failure(E) }
enum Expr { case num(Int); indirect case add(Expr, Expr) }   // indirect for recursive payloads
enum Never: Error {}   // uninhabited; used for impossible-error typed throws
```
- Exhaustive `switch` — compiler forces all cases; `@unknown default` for frozen-library evolution.
- **if/switch expressions** (Swift 5.9): `let x = if cond { 1 } else { 2 }`, `let y = switch v { case .a: "A" ... }`.
- Enums with raw values auto-synthesize `init?(rawValue:)` and `rawValue`.
- `Optional`, `Result` are enums — pattern matching is the idiomatic unwrap.

## 5. Structs, classes, properties

```swift
struct Point { var x = 0.0; let y: Double }   // memberwise init synthesized (internal)
class C { let id: UUID = UUID(); deinit { ... } }
```
- `let` = immutable binding; `var` = mutable. Struct `let` props immutable after init; class `let` props settable in init only.
- Property observers: `willSet`/`didSet` (not in init). `lazy var` computed once on first access (class-bound ok; thread-unsafe).
- Computed props `{ get set }` shorthand `{ get }` = read-only; `_modify`/`_read` yield for performance (rarely hand-written).
- Static vs class properties: `static` non-overridable, `class var` overridable in subclasses.
- Access control: `private` (file+type scope in Swift — `private` members accessible in same-file extensions of the type), `fileprivate` (whole file), `internal` (module default), `public` (module API, no subclassing unless `open`), `open` (subclassable outside module). `package` (Swift 5.9) = shared across packages in same package.
- Memberwise init is `internal` by default — add explicit `public init` for library types.

## 6. Error handling

```swift
do { try f() } catch let e as MyError { ... } catch { ... }   // catch-all required unless exhaustive typed throws
let result = try? f()            // Optional
let value = try! f()             // crash on throw — avoid
let r = Result { try f() }       // defer handling
```
- `Error` protocol: `LocalizedError` (errorDescription), `CustomNSError` (domain/code), `Equatable` conformance for testable errors.
- **Typed throws** (Swift 6): `func f() throws(MyError) -> Int` — do/catch can be exhaustive; `Result<T, MyError>` precision. `Never` = non-throwing function that "throws" unreachably.
- `rethrows`: function throws only if its function argument throws — preserves typed info.
- Errors are values; `throw` any `Error`. Prefer enum errors with associated context.

## 7. Ownership: consuming, borrowing, copy (noncopyable types)

```swift
func consume(_ x: consuming MyType)      // takes ownership; caller can't use x after
func borrow(_ x: borrowing MyType)       // guaranteed no mutation during call (default for class params too)
let y = x                                 // copy (for value types) or retain (class)
let z = consume x                         // explicit end-of-lifetime for class refs (optional)
```
- Noncopyable types: `struct FileHandle: ~Copyable` — moved not copied; enable with `borrowing`/`consuming` params, `each` packs. Used for RAII resources (file descriptors, continuations). Rarely hand-written; know it exists for stdlib APIs.
- `_eagerMove`, `_noEagerMove`, `copy` expr — underscored/experimental, mostly compiler-internal.

## 8. Property wrappers (language feature)

```swift
@propertyWrapper struct Clamped { var wrappedValue: Double { didSet { ... } } }
@Clamped var value = 0.0
```
- `@State`, `@Binding`, `@ObservedObject`, `@StateObject`, `@Environment`, `@AppStorage` are SwiftUI wrappers — mechanism: `wrappedValue` (get/set) + `projectedValue` (`$foo`).
- Custom wrappers get `$` projection via `projectedValue` — this is how two-way bindings are spelled.

## 9. Result builders & macros

- **Result builders** (Swift 5.4, SE-0289): transform statement sequences into values — `@ViewBuilder`, `@SceneBuilder`, custom `@resultBuilder` with `buildBlock`, `buildOptional`, `buildEither`, `buildArray`, `buildLimitedAvailability`, `buildFinalResult`, `buildPartialBlock` (incremental). Used by every SwiftUI DSL.
- **Macros** (Swift 5.9+): compile-time code generation.
  - Freestanding: `#Predicate`, `#stringify`, `#fileID`, custom `#externalMacro`.
  - Attached: `@Observable`, `@Model` (SwiftData), `@Generable` (Foundation Models), `@Entry`, `@attached(member)` — attached macros can add members, conformances, peers.
  - Declaration macros can't see function bodies (limitation); peer/member/extension macros can inspect.
- `@Observable` (Observation framework): macro adds observation tracking to a class — replaces `ObservableObject`/`@Published` for new code; works via `withObservationTracking`/`@Query`.
- `defaultIsolation(MainActor.self)` — Swift 6.2+: module-wide default actor isolation for the whole file/module.
- `@backDeployed(before:)` — stdlib attribute making new APIs available on older OS versions (that's how `if #available` isn't always needed for stdlib additions).

## 10. Attributes worth knowing

| Attribute | Effect |
|---|---|
| `@MainActor` | isolate type/func/property to main actor |
| `@available(iOS 17, macOS 14, *)` | API gating |
| `@discardableResult` | silence unused-result warning |
| `@inlinable`/`@usableFromInline` | cross-module optimization w/o library evolution |
| `@frozen`/`@usableFromInline` | ABI stability markers |
| `@_spi(Internal)` | SPI import for framework internals (e.g. `import SwiftUI._spi(Private)` — avoid in prod) |
| `@preconcurrency` | suppress Sendable warnings for pre-concurrency deps |
| `@_documentation(visibility:)` | doc-only visibility |
| `@unknown default` | future-proof exhaustive switch |
| `@diagnose(GroupID, as: error\|warning\|ignored, reason:)` | **Swift 6.4 (SE-0522)**: source-level warning control per scope |
| `~Sendable` | **Swift 6.4 (SE-0518)**: explicit non-Sendable suppression |
| `@Observable`, `@Model`, `@AppIntent`, etc. | macro attributes |

## 11. Swift 6.4 highlights (Sept 2026)

- **SE-0521**: `some P?` / `any P?` without parentheses.
- **SE-0522**: `@diagnose` attribute — promote/suppress specific warning groups lexically.
- **SE-0493**: `defer` supports `await`.
- **SE-0503**: `~Copyable`/`~Escapable` associated-type requirements.
- **SE-0518**: `~Sendable` explicit suppression.
- **SE-0508**: trailing closures after array literals.
- Typed-throws `Task {}` initializers now warn if thrown error isn't handled.
- `withTaskCancellationShield` — temporarily shield a task region from cancellation.
- Noncopyable-element `Array`/`InlineArray` without COW overhead; `Iterable` protocol.
- `Span` bridges to C++20 `std::span`; Java interop async/callback improvements.
- **Swift Build** is now SwiftPM's default build system (same build across Linux/macOS/Windows).
- **Subprocess 1.0** — stable cross-platform process spawning (`Subprocess.run`, async sequences for stdout/stderr).
- Embedded Swift: existential types + richer errors on microcontrollers.
- Wasm: JavaScriptKit bridging up to 40× faster; Wasm SDK from swift.org.

## 12. Swift 6 language mode — what changes vs Swift 5

- **Strict concurrency is on**: `Sendable` checking, actor isolation enforcement, main-actor-by-default inference for UI frameworks. Build warnings become errors in Swift 6 mode.
- `any` required for existentials: `let x: any Collection` (bare `Collection` as type is an error).
- `some` still for opaque returns.
- `@preconcurrency import` for not-yet-concurrency-safe modules.
- Region-based isolation (SE-0414): compiler proves non-overlapping access regions — fewer false sharing errors.
- Isolated conformances: `extension Foo: @MainActor SomeProtocol` or `isolated` witness params.
- `sending` parameter/result annotations for ownership transfer across isolation.
- Package access level, `if`/`switch` expressions, bare-slash regex all from 5.9 era now baseline.
- Migration: Xcode offers "Swift 5 → Swift 6" migrator; fix Sendable/actor warnings incrementally — don't silence with `@unchecked Sendable` unless you can prove safety.

## 13. Interop quick facts

- **Objective-C**: `@objc`, `@objcMembers`, `dynamic`; bridging headers for C in mixed targets (rare now); `NSObject` subclassing for delegates/KVO.
- **C/C++**: direct interop in Swift 5.9+ (`import` C++ modules with `-cxx-interoperability-mode`); `OpaquePointer`, `UnsafePointer`, `UnsafeMutablePointer`, `UnsafeBufferPointer`, `withUnsafeBytes`, `withContiguousStorageIfAvailable`.
- **C callbacks into Swift**: `@convention(c)` function pointers, `Unmanaged.passRetained/passUnretained` for bridging refs, `@escaping` + context pointer.
- **Unsafe pointers**: `UnsafeMutableRawPointer`, `assumingMemoryBound`, `bindMemory`, alignment rules — use only at system boundaries.

## 14. Common compile/runtime errors → cause

| Error/warning | Cause |
|---|---|
| `value of type 'X' has no member 'y'` | wrong type / optional not unwrapped / missing import |
| `cannot find 'X' in scope` | missing `import`, typo, or module not in target |
| `main actor-isolated property can not be referenced` | touching MainActor state off-actor — add `await`/`MainActor.run` or annotate |
| `sending 'x' risks causing data races` | non-Sendable crossing isolation — restructure or justify `@unchecked` |
| `expression is 'async' but is not marked with 'await'` | missing await; also means func must be `async` |
| `overlapping accesses` | exclusivity violation — modify same var in nested call (`sort` while reading self) |
| `variable used within its own initial value` | `let x = x + 1` shadowing trap — rename |
| `ambiguous type name` | type inference failure — annotate explicit type |
| `generic parameter 'T' could not be inferred` | compiler can't deduce — provide explicit generic or type annotation |
| Crash: `EXC_BAD_ACCESS`/`EXC_BREAKPOINT` | force-unwrap nil, out-of-bounds, `fatalError`, retain cycle isn't this (that's leak) |
| `Thread 1: EXC_BAD_INSTRUCTION` | Swift runtime trap (overflow, force unwrap, array bounds) |

## 15. Style & idioms

- `let` by default, `var` only when mutated. `guard` for early exits over nested `if let`.
- Prefer `first(where:)`/`contains(where:)` over manual loops for search; `map`/`compactMap`/`flatMap`/`reduce` for transforms; `sorted`, `filter`, `prefix`, `dropFirst`.
- `switch` over `if/else` chains for enums/multiple conditions.
- `private` by default for members; widen only as needed (internal default, public for API).
- Naming: types `UpperCamelCase`, members `lowerCamelCase`, protocols as nouns (`Collection`) or `-able/-ible`/`-ing` (`Equatable`, `Sendable`, `Decodable`).
- No force unwraps/`try!`/`as!` in production code — use `guard let`, `try?`+handle, `as?`+handle.
- `#if DEBUG` for dev-only code; `assert`/`precondition`/`fatalError` semantics: assert (debug only), precondition (always, optimized out only with -Ounchecked), fatalError (always).
- `some` for opaque params too in Swift 5.7+ (`func f(_ x: some Collection)`) — sugar for generic.
- `if let x, let y` multi-unwrap (Swift 5.7+); `guard let self` in closures.
- String switch on enum with associated values: `case .success(let v):` etc.
