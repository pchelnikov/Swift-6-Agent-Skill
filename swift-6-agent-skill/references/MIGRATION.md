# Migration Patterns: Old → New

This file maps deprecated or outdated Swift patterns to their modern replacements.
Use this when refactoring existing code or reviewing code that uses pre-6.0 idioms.

## Concurrency

### Global Actor Inference on Views

```swift
// OLD (Swift 5.x): @MainActor inferred from @StateObject
struct LogInView: View {
    @StateObject private var model = ViewModel()
    var body: some View { ... }
}

// NEW (Swift 6.0+): Explicit @MainActor required (SE-0401)
@MainActor
struct LogInView: View {
    @StateObject private var model = ViewModel()
    var body: some View { ... }
}
```

### Global Mutable State

```swift
// OLD: Bare global variables
var appConfig = AppConfig()

// NEW (Swift 6.0+): Must be concurrency-safe (SE-0412)
// Option 1: Make it a constant
let appConfig = AppConfig()
// Option 2: Isolate to an actor
@MainActor var appConfig = AppConfig()
// Option 3: Last resort — acknowledge unsafety
nonisolated(unsafe) var appConfig = AppConfig()
```

### TaskGroup Type Parameter

```swift
// OLD (Swift 6.0):
await withTaskGroup(of: String.self) { group in
    group.addTask { "Hello" }
    ...
}

// NEW (Swift 6.1+): Type inferred (SE-0442)
await withTaskGroup { group in
    group.addTask { "Hello" }
    ...
}
```

### Opting Out of Actor Isolation

```swift
// OLD: Mark each member nonisolated individually
struct App: DataStoring {
    nonisolated let controller = DataController()
    nonisolated func process() { ... }
}

// NEW (Swift 6.1+): Apply nonisolated at the type level (SE-0449)
nonisolated struct App: DataStoring {
    let controller = DataController()
    func process() { ... }
}
```

### MainActor Everywhere

```swift
// OLD: Annotate every type with @MainActor manually
@MainActor struct SettingsView: View { ... }
@MainActor class UserManager { ... }
@MainActor struct ProfileView: View { ... }

// NEW (Swift 6.2+): Use default isolation per module (SE-0466)
// Add -default-isolation MainActor to compiler flags
// Then only annotate exceptions:
struct SettingsView: View { ... }       // Automatically MainActor
class UserManager { ... }              // Automatically MainActor
nonisolated struct BackgroundWorker { } // Explicit opt-out
```

### Nonisolated Async Behavior

```swift
// OLD (Swift 5.7–6.1): Nonisolated async hops off caller's actor (SE-0338)
// fetchData() runs on generic executor even if called from @MainActor
func fetchData() async -> Data { ... }

// NEW (Swift 6.2+): Stays on caller's actor by default (SE-0461)
// fetchData() runs on MainActor if called from MainActor context
func fetchData() async -> Data { ... }

// To get old behavior, use @concurrent:
@concurrent func fetchData() async -> Data { ... }
```

### Regular Task vs Immediate Task

```swift
// OLD: Task starts at next scheduling opportunity
Task { updateUI() }  // Queued, not immediate

// NEW (Swift 6.2+): Task starts synchronously (SE-0472)
Task.immediate { updateUI() }  // Runs now, until first await
```

### Deinit in Actor-Isolated Classes

```swift
// OLD: Cannot access actor-isolated state in deinit
@MainActor class Session {
    let user: User
    deinit {
        // user.isLoggedIn = false  // Error: non-isolated deinit
    }
}

// NEW (Swift 6.2+): isolated deinit (SE-0371)
@MainActor class Session {
    let user: User
    isolated deinit {
        user.isLoggedIn = false  // Safe: runs on MainActor
    }
}
```

## Error Handling

### Catch-All for Known Error Types

```swift
// OLD: Required catch-all even with exhaustive cases
do {
    try copier.copy(count: 101)
} catch CopierError.outOfPaper {
    print("No paper")
} catch {
    // Required but never reached
}

// NEW (Swift 6.0+): Typed throws eliminates catch-all (SE-0413)
// func copy(count:) throws(CopierError)
do {
    try copier.copy(count: 101)
} catch CopierError.outOfPaper {
    print("No paper")
}
// No catch-all needed — compiler knows all cases
```

## Types and Data

### Weak References in Sendable Types

```swift
// OLD: weak var prevents Sendable conformance
final class Session {
    weak var user: User?  // Cannot be Sendable
}

// NEW (Swift 6.2+): weak let enables Sendable (SE-0481)
final class Session: Sendable {
    weak let user: User?  // Immutable weak — Sendable OK
}
```

### Fixed-Size Collections

```swift
// OLD: Tuples for fixed-size data, no subscripting
let colors = ("red", "green", "blue")
// colors[0]  // Not allowed on tuples

// NEW (Swift 6.2+): InlineArray (SE-0453)
var colors: InlineArray<3, String> = ["red", "green", "blue"]
colors[0]  // "red" — subscript works
```

## String Interpolation

### Optional Values with Cross-Type Defaults

```swift
// OLD: Nil coalescing requires same type
var age: Int? = nil
print("Age: \(age ?? 0)")            // Must default to Int
// print("Age: \(age ?? "Unknown")") // Compile error

// NEW (Swift 6.2+): Default value in interpolation (SE-0477)
print("Age: \(age, default: "Unknown")")  // Cross-type OK
```

## Key Paths

### Static Properties

```swift
// OLD: No key path to static properties
// \WarpDrive.maximumSpeed  // Error if maximumSpeed is static

// NEW (Swift 6.1+): Metatype key paths (SE-0438)
let path = \WarpDrive.Type.maximumSpeed
```

### Methods

```swift
// OLD: Key paths only for properties
let caps = strings.map { $0.uppercased() }

// NEW (Swift 6.2+): Method key paths (SE-0479)
let caps = strings.map(\.uppercased())
```

## Imports

### Preventing Transitive Import Leaks

```swift
// OLD: Import in one file could leak to others
// FileA.swift: import Maps → toRadians() available everywhere

// NEW (Swift 6.1+): Enable MemberImportVisibility flag (SE-0444)
// Each file must explicitly import what it uses
```

## Testing

### Test Names

```swift
// OLD: Duplicate name in annotation and function
@Test("Strip HTML tags from string")
func stripHTMLTagsFromString() { ... }

// NEW (Swift 6.2+): Raw identifiers (SE-0451)
@Test func `Strip HTML tags from string`() { ... }
```

### Error Validation in Tests

```swift
// OLD (deprecated): Dual trailing closure
#expect { try playGame(at: 22) } throws: { error in
    guard let e = error as? GameError else { return false }
    return e == .disallowedTime
}

// NEW (Swift 6.1+): Returned error (ST-0006)
let error = #expect(throws: GameError.self) {
    try playGame(at: 22)
}
#expect(error == .disallowedTime)
```

### Testing Fatal Errors

```swift
// OLD: No way to test precondition/fatalError without crashing test runner
// Various hacks with custom assertion handlers

// NEW (Swift 6.2+): Exit tests (ST-0008)
await #expect(processExitsWith: .failure) {
    dice.roll(sides: 0)  // precondition failure — caught
}
```

### Observable Changes Outside SwiftUI

```swift
// OLD: Combine publisher + sink, or manual KVO
let cancellable = player.$score.sink { print($0) }

// NEW (Swift 6.2+): Observations (SE-0475)
let scores = Observations { player.score }
for await score in scores { print(score) }
```

## Synchronization

### Thread-Safe Shared State

```swift
// OLD: NSLock, os_unfair_lock, DispatchQueue.sync
let lock = NSLock()
lock.lock()
counter += 1
lock.unlock()

// NEW (Swift 6.0+): Mutex (SE-0433)
import Synchronization
let counter = Mutex(0)
counter.withLock { $0 += 1 }
```

### Lock-Free Atomic Access

```swift
// OLD: OSAtomicIncrement32, or manual lock for flags
var isReady: Bool { // Thread-unsafe
    didSet { notify() }
}

// NEW (Swift 6.0+): Atomic (SE-0410)
import Synchronization
let isReady = Atomic(false)
isReady.store(true, ordering: .releasing)
```

## Types

### Fixed-Size Collections (Syntax)

```swift
// OLD (Swift 6.2): Full type name
var colors: InlineArray<3, String> = ["red", "green", "blue"]

// NEW (Swift 6.2+): Sugar syntax (SE-0483)
var colors: [3 of String] = ["red", "green", "blue"]
```

### Extensible Package Enums

```swift
// OLD: Adding a case to a public enum = SemVer-major break
// Workaround: Use structs with static lets, losing switch ergonomics

// NEW (Swift 6.2.3+): Extensible by default (SE-0487)
public enum APIError: Error {
    case notFound
    case unauthorized
    // Can add new cases in minor versions
}

// Freeze only when the case list is truly fixed:
@frozen public enum Direction { case north, south, east, west }

// Consumers must add @unknown default:
switch error {
case .notFound: handle404()
case .unauthorized: handle401()
@unknown default: handleUnknown(error)
}
```

## Debugging

### Custom LLDB Formatting

```swift
// OLD: Only po uses debugDescription; p shows raw fields
struct Org: CustomDebugStringConvertible {
    var id: String; var name: String
    var debugDescription: String { "#\(id) \(name)" }
}

// NEW (Swift 6.0+): @DebugDescription (SE-0440) — works with p too
@DebugDescription
struct Org: CustomDebugStringConvertible {
    var id: String; var name: String
    var debugDescription: String { "#\(id) \(name)" }
}
```

## Protocols and Generics

### Retroactive Conformance

```swift
// OLD: Silent retroactive conformance — potential ABI/behavior conflicts
extension URLSession: MyProtocol { }

// NEW (Swift 6.0+): Compiler warns; use @retroactive (SE-0364)
extension URLSession: @retroactive MyProtocol { }
```

### AsyncSequence Typed Throws

```swift
// OLD: AsyncSequence had experimental Failure handling
// Cannot distinguish throwing from non-throwing async sequences generically

// NEW (Swift 6.0+): Failure associated type + typed throws (SE-0421)
// for await item in nonThrowingStream { } — no try needed
// for try await item in throwingStream { } — error type is known
```

## Strings

### Safe Encoding Conversion

```swift
// OLD: String(bytes:encoding:) with unclear failure semantics
let str = String(bytes: data, encoding: .utf8) // NSString-based

// NEW (Swift 6.0+): String(validating:as:) (SE-0405)
let str = String(validating: data, as: UTF8.self) // Returns nil on invalid
```
