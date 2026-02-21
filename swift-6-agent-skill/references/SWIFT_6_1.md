# Swift 6.1 Features Reference

Released: Spring 2025. Compiler: Xcode 16.3+.

## Trailing Commas in Lists (SE-0439)

Trailing commas are now permitted in all comma-separated lists bounded by `()`, `[]`, or `<>`.

```swift
func add<T: Numeric,>(_ a: T, _ b: T,) -> T { a + b }

let range = message.range(
    of: "impossible",
    options: .caseInsensitive,  // Can comment out this line without breaking code
)
```

**Restriction**: Does not apply to unbounded lists such as enum cases.

## Metatype Key Paths (SE-0438)

Key paths now support static properties via `.Type`.

```swift
struct WarpDrive {
    static let maximumSpeed = 9.975
    var currentSpeed = 8.0
}

let instancePath = \WarpDrive.currentSpeed           // KeyPath<WarpDrive, Double>
let staticPath   = \WarpDrive.Type.maximumSpeed       // KeyPath<WarpDrive.Type, Double>

// With type annotation:
let typed: KeyPath<WarpDrive.Type, Double> = \.maximumSpeed
```

## TaskGroup Child Type Inference (SE-0442)

The `of:` parameter in `withTaskGroup` and `withThrowingTaskGroup` can now be omitted.

```swift
// Before (Swift 6.0):
let result = await withTaskGroup(of: String.self) { group in ... }

// After (Swift 6.1+):
let result = await withTaskGroup { group in
    group.addTask { "Hello" }
    group.addTask { "World" }
    // ...type inferred from child tasks
}
```

## nonisolated on Types (SE-0449)

Apply `nonisolated` to protocols, structs, classes, and enums to opt out of inherited global actor isolation.

```swift
@MainActor
protocol DataStoring {
    var controller: DataController { get }
}

// Without nonisolated: App inherits @MainActor from DataStoring
// With nonisolated: App opts out, must use await for actor-isolated access
nonisolated struct App: DataStoring {
    let controller = DataController()
    init() async {
        await controller.load()
    }
}
```

## Member Import Visibility (SE-0444)

Requires explicit imports per file. Enable via the `MemberImportVisibility` upcoming feature flag.

Previously, importing a module in one file could leak extension methods to all files in the project. With this flag enabled, each file must import the modules it uses.

**Migration**: Add missing `import` statements where code relied on transitive imports.

## Diagnostic Groups (SE-0443)

Fine-grained control over compiler warning/error levels.

**Setup**: Add `-print-diagnostic-groups` to Swift compiler custom flags.

```
// Warnings now include group names:
// 'sonomaFunction()' was deprecated [DeprecatedDeclaration]

// Upgrade specific warnings to errors:
// -Werror DeprecatedDeclaration

// Keep specific warnings as warnings even with treat-warnings-as-errors:
// -Wwarning DeprecatedDeclaration
```

**Note**: Flag order matters. Xcode applies build settings in its own order; you may need `-warnings-as-errors` positioned manually in Other Swift Flags.

## Language Mode Terminology (SE-0441)

Swift now formally distinguishes **compiler version** from **language mode**. The term "Swift language mode" describes which version of the language rules are active (e.g., Swift 6 language mode running on the Swift 6.1 compiler).

## Swift Testing Improvements

### Range-Based Confirmations (ST-0005)

```swift
await confirmation(expectedCount: 5...10) { confirm in
    for await _ in loader { confirm() }
}

// Also: open-ended ranges
await confirmation(expectedCount: 5...) { confirm in ... }

// Disallowed: ...10 (ambiguous start point)
```

### Returned Errors from #expect(throws:) (ST-0006)

```swift
// Deprecated dual-trailing-closure form:
#expect { try playGame(at: 22) } throws: { error in ... }

// New (Swift 6.1+): returns the error for separate validation
let error = #expect(throws: GameError.self) {
    try playGame(at: 22)
}
#expect(error == .disallowedTime)
```

### Test Scoping Traits (ST-0007)

Concurrency-safe test environment configuration via `TestScoping` protocol.

```swift
struct DefaultPlayerTrait: TestTrait, TestScoping {
    func provideScope(
        for test: Test, testCase: Test.Case?,
        performing function: () async throws -> Void
    ) async throws {
        let player = Player(name: "Natsuki Subaru")
        try await Player.$current.withValue(player) {
            try await function()
        }
    }
}

extension Trait where Self == DefaultPlayerTrait {
    static var defaultPlayer: Self { Self() }
}

@Test(.defaultPlayer) func welcomeScreenShowsName() {
    let result = createWelcomeScreen()
    #expect(result.contains("Natsuki Subaru"))
}
```

## Additional Changes

- **SE-0387**: Improved cross-compilation support (e.g., build Linux binaries on macOS Apple Silicon).
- **SE-0436**: `@objc @implementation extension` can replace Objective-C `@implementation` blocks, enabling incremental migration of Obj-C classes to Swift one category at a time.
- **SE-0445**: `String.Index` now has improved debug printing, showing the offset and encoding.
- **SE-0450**: Swift packages can provide optional traits for feature toggling via `Package.swift` configuration.
