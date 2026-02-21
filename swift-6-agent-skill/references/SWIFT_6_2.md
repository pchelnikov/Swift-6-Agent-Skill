# Swift 6.2 Features Reference

Expected: WWDC 2025. Compiler: Xcode 17+.

## Default Actor Isolation Inference (SE-0466)

Modules can opt into running all code on the main actor by default.

**Enable**: Add `-default-isolation MainActor` to compiler flags.

```swift
// With default MainActor isolation, no annotations needed:
class DataController {
    func load() { }
    func save() { }
}

struct App {
    let controller = DataController()
    init() {
        controller.load()  // No await needed — both on MainActor
    }
}
```

Key points:
- Applied per-module. External modules retain their own isolation.
- `URLSession.shared.data(from:)` and similar still run on their own tasks — no blocking.
- Opt out individual items with `nonisolated` or `@concurrent`.
- Part of the "Improving the approachability of data-race safety" vision.

## Raw Identifiers (SE-0451)

Backtick-wrapped identifiers can now contain spaces, numbers as leading characters, and operator characters (mixed with non-operators).

```swift
// Human-readable test names
@Test func `Strip HTML tags from string`() { ... }

// Numeric enum cases
enum HTTPError: String {
    case `401` = "Unauthorized"
    case `404` = "Not Found"
    case `500` = "Internal Server Error"
}

// Usage requires qualification or backticks
let error = HTTPError.401
switch error {
case .`401`, .`404`: print("Client error")
default: print("Server error")
}
```

**Rule**: Raw identifiers may contain operator characters but cannot consist *only* of operator characters.

## Default Values in String Interpolation (SE-0477)

Optionals in string interpolation can now specify a default of any type.

```swift
var age: Int? = nil
print("Age: \(age, default: "Unknown")")  // Cross-type default — works!

// Previously required same-type nil coalescing:
// print("Age: \(age ?? 0)")           // OK but forced to Int
// print("Age: \(age ?? "Unknown")")   // Would not compile
```

## enumerated() Conforms to Collection (SE-0459)

`enumerated()` now returns a `Collection`, enabling direct use with SwiftUI.

```swift
List(names.enumerated(), id: \.offset) { values in
    Text("User \(values.offset + 1): \(values.element)")
}
```

Also enables constant-time operations like `(1000..<2000).enumerated().dropFirst(500)`.

## Method and Initializer Key Paths (SE-0479)

Key paths now support methods (in addition to properties and subscripts).

```swift
let uppercased = strings.map(\.uppercased())      // Invoke method
let functions = strings.map(\.uppercased)          // Get uninvoked reference

// Disambiguate overloads with argument labels
let prefixUpTo = \Array<String>.prefix(upTo:)
let prefixThrough = \Array<String>.prefix(through:)
```

**Limitation**: No key paths to `async` or `throws` methods.

## Strict Memory Safety Checking (SE-0458)

Opt-in `@safe`/`@unsafe` attributes for auditing unsafe code.

```swift
let name: String?
unsafe print(name.unsafelyUnwrapped)  // Must acknowledge with `unsafe` keyword
```

Similar to `try`/`await` — a compile-time acknowledgement for humans.

## Backtrace API (SE-0419)

Capture and symbolicate call stacks at runtime.

```swift
import Runtime

if let frames = try? Backtrace.capture().symbolicated()?.frames {
    print(frames)  // Function names, files, line numbers
}
```

## weak let (SE-0481)

Immutable weak references. Cannot be reassigned, but can be destroyed. Enables `Sendable` conformance.

```swift
final class Session: Sendable {
    weak let user: User?       // Would not be Sendable with weak var
    init(user: User?) { self.user = user }
}
```

Cannot reassign: `session.user = nil` and `session.user? = User()` are compile errors.

## Transactional Observation (SE-0475)

`Observations` struct provides an `AsyncSequence` of `@Observable` changes.

```swift
@Observable class Player { var score = 0 }

let player = Player()
let scores = Observations { player.score }

for await score in scores {
    print(score)  // Emits initial value + all changes
}
```

Notes:
- Emits initial value on subscription.
- Coalesces rapid changes (may skip intermediate values).
- Runs potentially forever — use on a separate task.
- Set observed value to `nil` (if optional) to end iteration.

## Global-Actor Isolated Conformances (SE-0470)

Restrict protocol conformance to a specific actor.

```swift
@MainActor
class User: @MainActor Equatable {
    var id: UUID
    var name: String
    static func ==(lhs: User, rhs: User) -> Bool { lhs.id == rhs.id }
}
```

Without `@MainActor` on `Equatable`, the `==` method could run off the main actor, violating the class's isolation.

## Immediate Tasks (SE-0472)

Tasks that start executing synchronously before the first suspension point.

```swift
Task.immediate {
    print("Runs before anything else queued")
    await someAsyncWork()  // May suspend here
}

// Also in task groups:
group.addImmediateTask { ... }
group.addImmediateTaskUnlessCancelled { ... }
```

Difference: Regular `Task` is queued; `Task.immediate` runs synchronously until first `await`.

## Nonisolated Async Runs on Caller's Actor (SE-0461)

Nonisolated async functions now inherit the caller's actor by default (changed from SE-0338 behavior).

```swift
struct Measurements {
    func fetchLatest() async throws -> [Double] {
        // In Swift 6.2: runs on caller's actor (e.g., MainActor)
        // In Swift <6.2: would hop off to a background executor
        let (data, _) = try await URLSession.shared.data(from: url)
        return try JSONDecoder().decode([Double].self, from: data)
    }
}

@MainActor
struct WeatherStation {
    let measurements = Measurements()
    func getAverage() async throws -> Double {
        let readings = try await measurements.fetchLatest()  // Stays on MainActor
        return readings.reduce(0, +) / Double(readings.count)
    }
}
```

**Opt out**: Mark the function `@concurrent` to restore the old hop-off behavior.

## Isolated Synchronous Deinit (SE-0371)

Actor-isolated classes can mark `deinit` as `isolated` to safely access non-Sendable state.

```swift
@MainActor
class Session {
    let user: User  // Non-Sendable

    init(user: User) { user.isLoggedIn = true }

    isolated deinit {
        user.isLoggedIn = false  // Safe: runs on MainActor
    }
}
```

## Task Priority Escalation APIs (SE-0462)

Detect and respond to priority changes.

```swift
let fetcher = Task(priority: .medium) {
    try await withTaskPriorityEscalationHandler {
        // do work
    } onPriorityEscalated: { old, new in
        print("Escalated from \(old) to \(new)")
    }
}

// Manual escalation
fetcher.escalatePriority(to: .high)
```

## Task Naming (SE-0469)

```swift
let task = Task(name: "FetchUser") { ... }
print(Task.name ?? "Unknown")

// In task groups:
group.addTask(name: "Stories \(i)") { ... }
```

## InlineArray (SE-0453) and Sugar Syntax (SE-0483)

Fixed-size array with compile-time count. Uses integer generic parameters (SE-0452).

```swift
// Full type name:
var names: InlineArray<4, String> = ["Moon", "Mercury", "Mars", "Tuxedo Mask"]

// Sugar syntax (SE-0483, Swift 6.2+):
var names2: [4 of String] = ["Moon", "Mercury", "Mars", "Tuxedo Mask"]

// Inferred:
var names3: InlineArray = ["Moon", "Mercury", "Mars", "Tuxedo Mask"]

names[2] = "Jupiter"  // Subscript OK

// No Sequence/Collection conformance — iterate via indices:
for i in names.indices { print(names[i]) }

// No append() or remove(at:) — fixed size
```

**Prefer the sugar `[N of T]` syntax** in new code for readability. The full `InlineArray<N, T>` name is equivalent.

## Regex Lookbehind Assertions (SE-0448)

```swift
let string = "Jacket $100, shoes $59.99"
let regex = /(?<=\$)\d+(?:\.\d{2})?/
for match in string.matches(of: regex) {
    print(match.output)  // "100", "59.99"
}
```

## Swift Testing Improvements

### Exit Tests (ST-0008)

Test code paths that call `precondition`, `fatalError`, or `exit`.

```swift
@Test func invalidDiceRollsFail() async throws {
    let dice = Dice()
    await #expect(processExitsWith: .failure) {
        let _ = dice.roll(sides: 0)  // Would crash — now caught
    }
}
```

### Attachments (ST-0009)

Attach data to tests for post-failure debugging.

```swift
struct Character: Codable, Attachable {
    var id = UUID()
    var name: String
}

@Test func nameIsCorrect() {
    let result = makeCharacter()
    #expect(result.name == "Rem")
    Attachment.record(result, named: "Character")
}
```

Supported out of the box: `String`, `Data`, `Encodable`. No image support yet.

### ConditionTrait.evaluate() (ST-0010)

Evaluate test conditions programmatically outside `@Test`.

```swift
let trait = ConditionTrait.disabled(if: TestManager.inSmokeTestMode)
if try await trait.evaluate() {
    print("Smoke test mode active")
}
```

## Span and Memory Safety Types

### Span (SE-0447)

Safe, non-owning view over contiguous memory. Maintains memory safety by ensuring the memory remains valid while you're using it — checked at compile time, no runtime overhead.

```swift
func process(_ data: Span<UInt8>) {
    for byte in data { print(byte) }
}
```

### MutableSpan (SE-0467)

Mutable version of `Span` for in-place modification of contiguous memory.

### OutputSpan (SE-0485)

Output variant for writing to uninitialized memory safely.

### UTF8Span (SE-0464)

Safe view over UTF-8 encoded data with string processing capabilities.

## Feature Adoption Tooling (SE-0486)

Migration tooling for upcoming language features. The compiler produces warnings identifying code patterns that will change behavior, and provides fix-its to automate code updates. Helps stage in upcoming features like `NonisolatedNonsendingByDefault` incrementally.

## Additional Changes

- **SE-0446**: Non-escapable types that cannot outlive their creating scope.
- **SE-0452**: Integer generic parameters (foundation for `InlineArray`).
- **SE-0456**: `.span` properties on stdlib container types for safe memory access.
- **SE-0457**: `Duration.totalAttoseconds` exposed as `Int128`.
- **SE-0463**: Objective-C completion handler parameters default to `@Sendable`.
- **SE-0465**: Non-escapable stdlib primitives — extends non-escapable types to standard library.
- **SE-0468**: `AsyncStream.Continuation` conforms to `Hashable` and `Equatable`.
- **SE-0471**: `SerialExecutor.isIsolated` for synchronous isolation checking.
- **SE-0473**: `SuspendingClock`/`ContinuousClock` expose their start point (zero time).
- **SE-0474**: Yielding accessors for copy-free read/write.
- **SE-0476**: `@abi` attribute for ABI-stable library evolution without breaking changes.
- **SE-0480**: Diagnostic group levels configurable in Swift packages via `treatWarning(_:as:)` and `treatAllWarnings(as:)` in `SwiftSetting`.
- **SE-0482**: SwiftPM static library binary targets on non-Apple platforms.
- **SE-0488**: Extracting pattern for noncopyable types.
