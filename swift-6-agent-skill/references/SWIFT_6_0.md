# Swift 6.0 Features Reference

Released: September 2024. Compiler: Xcode 16+. Language mode: Swift 6.

## Complete Concurrency Checking (Enabled by Default)

Swift 6 enables strict concurrency checking by default. The compiler uses **isolation regions (SE-0414)** to prove data-race safety, dramatically reducing false-positive warnings from Swift 5.10.

### Key changes

- **Isolation regions**: The compiler tracks value ownership across regions. Non-Sendable values used safely within a single region no longer trigger warnings.
- **`sending` keyword (SE-0430)**: Marks parameters/return values that transfer between isolation regions.
- **Actor inference removed for property wrappers (SE-0401)**: `@StateObject` and `@ObservedObject` no longer infer `@MainActor` on the enclosing view. Explicitly annotate views that need it.
- **Global variables must be concurrency-safe (SE-0412)**: Use `let`, restrict to a global actor (`@MainActor`), or mark `nonisolated(unsafe)` as a last resort.
- **Default value isolation (SE-0411)**: Function default values inherit the isolation of their enclosing function.
- **Async caller isolation (SE-0420)**: Async functions can be isolated to the same actor as their caller.

### Syntax examples

```swift
// SE-0414: No warning — compiler proves user stays in one region
class User { var name = "Anonymous" }

struct ContentView: View {
    var body: some View {
        Text("Hello").task {
            let user = User()
            await loadData(for: user) // Safe: user doesn't escape region
        }
    }
    func loadData(for user: User) async { }
}

// SE-0401: Explicit @MainActor required in Swift 6
@MainActor
struct LogInView: View {
    @StateObject private var model = ViewModel()
    var body: some View {
        Button("Log In", action: model.authenticate)
    }
}

// SE-0412: Global variable safety
struct XWing {
    @MainActor static var sFoilsAttackPosition = true  // Actor-isolated
}
struct WarpDrive {
    static let maximumSpeed = 9.975  // Constant — safe
}
nonisolated(unsafe) var legacyCache = [String]()  // Last resort
```

## Synchronization Library: Mutex and Atomics

### Mutex (SE-0433)

Thread-safe access to shared mutable state via a scoped locking API.

```swift
import Synchronization

let counter = Mutex(0)

// Access the value exclusively
counter.withLock { value in
    value += 1
}
```

Key points:
- `Mutex<Value>` is `~Copyable` and generic over `~Copyable` values.
- Use `withLock(_:)` for exclusive access — no manual lock/unlock.
- Prefer `Mutex` over `NSLock` or `os_unfair_lock` for new Swift code.
- For simple counters or flags, consider `Atomic` instead (lower overhead, no blocking).

### Atomics (SE-0410)

Lock-free concurrent access for types conforming to `AtomicRepresentable`.

```swift
import Synchronization

let flag = Atomic(false)
flag.store(true, ordering: .releasing)
let value = flag.load(ordering: .acquiring)

// Compare and exchange
let (exchanged, original) = flag.compareExchange(
    expected: true,
    desired: false,
    ordering: .acquiringAndReleasing
)
```

Key points:
- `Atomic<Value>` requires `Value: AtomicRepresentable`. Built-in conformances for integer types, `Bool`, `Optional` where `Wrapped: AtomicOptionalRepresentable`, and raw pointers.
- Memory ordering: `.relaxed`, `.acquiring`, `.releasing`, `.acquiringAndReleasing`, `.sequentiallyConsistent`.
- No runtime overhead beyond the atomic instruction itself — no locks, no blocking.

## Typed Throws (SE-0413)

Functions can declare the exact error type they throw.

```swift
enum CopierError: Error { case outOfPaper }

struct Photocopier {
    var pagesRemaining: Int

    mutating func copy(count: Int) throws(CopierError) {
        guard count <= pagesRemaining else { throw .outOfPaper }
        pagesRemaining -= count
    }
}

// Caller: no catch-all needed
do {
    var copier = Photocopier(pagesRemaining: 100)
    try copier.copy(count: 101)
} catch CopierError.outOfPaper {
    print("Refill paper")
}
```

Key rules:
- `throws(SomeError)` — exactly one error type.
- `throws(any Error)` ≡ `throws`.
- `throws(Never)` ≡ non-throwing.
- Cannot write `throws(A, B)`.
- Best for internal/embedded code; use untyped `throws` in public library APIs.

## @DebugDescription Macro (SE-0440)

Generates LLDB type summaries from `debugDescription`, enabling custom formatting in `p` command and Xcode's variables view without executing code.

```swift
@DebugDescription
struct Organization: CustomDebugStringConvertible {
    var id: String
    var name: String
    var manager: Person

    var debugDescription: String {
        "#\(id) \(name) [\(manager.name)]"
    }
}
// (lldb) p myOrg → "#100 Worldwide Travel [Jonathan Swift]"
```

Key points:
- Only supports simple string interpolations of stored properties.
- Falls back gracefully if `debugDescription` is too complex for the macro.
- Use alongside `CustomDebugStringConvertible` for both `po` and `p` coverage.

## Usability of Global-Actor-Isolated Types (SE-0434)

Stored properties of `Sendable` type in a global-actor-isolated type are implicitly nonisolated.

```swift
@MainActor
class UserAccount {
    nonisolated var profile: UserProfile  // Sendable stored property — accessible without await
    var mutableState: [String] = []       // Non-Sendable — remains actor-isolated

    nonisolated init(profile: UserProfile) {
        self.profile = profile
    }
}
```

Key points:
- Only applies to stored properties with `Sendable` type.
- Wrapped properties and `lazy` properties remain isolated (they are computed).
- Enables synchronous access to `Sendable` data from outside the actor.

## @isolated(any) Function Types (SE-0431)

Function types can dynamically carry the isolation of their caller.

```swift
func run(_ operation: @isolated(any) () async -> Void) {
    // The function's isolation is determined at runtime
    await operation()
}
```

Key points:
- Enables APIs that accept closures with arbitrary isolation (not just `@Sendable`).
- Used internally by structured concurrency APIs and task groups.
- Most relevant for library authors building concurrency primitives.

## Noncopyable Type Upgrades

### Generics Support (SE-0427)

All types implicitly conform to `Copyable`. Opt out with `~Copyable`. Noncopyable types now work with generics and protocols marked `~Copyable`.

### Stdlib Primitives (SE-0437)

`Optional` and `Result` can now wrap noncopyable types:

```swift
struct UniqueResource: ~Copyable { var handle: Int }
var maybeResource: Optional<UniqueResource> = UniqueResource(handle: 42)
```

### Partial Consumption (SE-0429)

Noncopyable structs without deinitializers support partial consumption.

### Borrowing in Switch (SE-0432)

Pattern matching borrows noncopyable values with `where` clauses.

```swift
struct Message: ~Copyable {
    var agent: String
    private var message: String
    consuming func read() { print("\(agent): \(message)") }
}
```

## AsyncSequence Generics (SE-0421)

`AsyncSequence` now uses a `Failure` associated type and typed throws, replacing the previous experimental mechanism.

```swift
// AsyncSequence protocol now has:
// associatedtype Failure: Error = Never
// func next() async throws(Failure) -> Element?

// Non-throwing async sequences have Failure == Never
// Throwing async sequences specify the error type
```

Key points:
- Enables generic code to distinguish throwing from non-throwing async sequences.
- `for await` on a non-throwing sequence doesn't require `try`.
- Maps directly to typed throws (`throws(Failure)`).

## KeyPath Function Subtyping (SE-0416)

Key path expressions can be used wherever functions are expected.

```swift
let names = users.map(\.name)       // Key path as function
let adults = users.filter(\.isAdult) // Key path as predicate
```

Previously required explicit closure: `.map { $0.name }`.

## count(where:) (SE-0220)

Single-pass count with predicate on any `Sequence`.

```swift
let scores = [100, 80, 85]
let passCount = scores.count { $0 >= 85 } // 2
```

## Pack Iteration (SE-0408)

Loop over parameter packs introduced in Swift 5.9.

```swift
func == <each Element: Equatable>(
    lhs: (repeat each Element),
    rhs: (repeat each Element)
) -> Bool {
    for (left, right) in repeat (each lhs, each rhs) {
        guard left == right else { return false }
    }
    return true
}
```

## RangeSet and Noncontiguous Collection Operations (SE-0270)

```swift
let topIndices = results.indices { $0.score >= 85 }
for result in results[topIndices] {  // DiscontiguousSlice
    print(result.student)
}
```

`RangeSet` supports `union()`, `intersection()`, `isSuperset(of:)`.

## Access-Level Imports (SE-0409)

```swift
internal import Transactions  // Transactions API not exposed to consumers
public import CoreModule      // Explicitly re-exported
```

Default in Swift 6 mode: `internal`. Default in Swift 5 mode: `public`.

## String Validating Initializers (SE-0405)

Safe string creation from encoded data with explicit encoding:

```swift
let data: [UInt8] = [0x48, 0x65, 0x6C, 0x6C, 0x6F]
let str = String(validating: data, as: UTF8.self)  // Optional<String>
```

## Retroactive Conformance Warning (SE-0364)

Compiler warns when you add a protocol conformance to a type you don't own in a module you don't own. Suppress with `@retroactive`:

```swift
extension SomeExternalType: @retroactive SomeExternalProtocol { }
```

## Int128 / UInt128 (SE-0425)

```swift
let big: Int128 = 170_141_183_460_469_231_731_687_303_715_884_105_727
```

## BitwiseCopyable (SE-0426)

Automatic for most structs/enums. Disabled for `public`/`package` types unless `@frozen`. Opt out with `~BitwiseCopyable`.

## Additional Changes

- **SE-0415**: `@attached(body)` macros for synthesizing and augmenting function implementations.
- **SE-0417**: Task executor preference — `withTaskExecutorPreference` for controlling which executor runs a task.
- **SE-0418**: Automatic `Sendable` inference for methods and key path literals.
- **SE-0422**: Expression macros as caller-side default arguments.
- **SE-0423**: Dynamic actor isolation enforcement from non-strict-concurrency contexts.
- **SE-0424**: Custom isolation checking for `SerialExecutor` — `checkIsolated()` method.
- **SE-0428**: Resolve distributed actor protocols — enables distributed actors to conform to protocols.
- **SE-0435**: Per-target Swift language version setting in SwiftPM.
- **SE-0301**: Package editing commands in SwiftPM CLI.
