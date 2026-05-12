# Swift 6.3 Features Reference

Released: March 2026. Compiler: Swift 6.3+.

Swift 6.3 focuses on C interoperability, module name disambiguation, library
optimization control, embedded/systems programming hooks, and small developer
experience improvements.

## C Compatible Functions and Enums (SE-0495)

Use `@c` to expose Swift global functions and integer-backed enums to C.
This formalizes the common `@_cdecl` use case with type checking and generated
C header support.

```swift
@c
func checksum(_ bytes: UnsafePointer<UInt8>, count: CInt) -> UInt32 {
    // Swift implementation callable from C
    0
}

@c(MyLibrary_checksum)
func checksumWithCName(_ bytes: UnsafePointer<UInt8>, count: CInt) -> UInt32 {
    0
}

@c
enum ParserState: CInt {
    case idle
    case parsing
    case failed
}
```

Key points:
- `@c` functions use the C calling convention.
- Function signatures must use C-representable types.
- `@c` enums must have a C-compatible integer raw type.
- Generated compatibility headers include declarations for C clients.
- Adding or removing `@c` on a function is ABI breaking.

### `@c @implementation`

Use `@c @implementation` when a C header already declares the function and Swift
provides the implementation.

```c
// MyLibrary.h
int mirror(int value);
```

```swift
@c @implementation
func mirror(_ value: CInt) -> CInt {
    value
}
```

The compiler checks that the Swift implementation matches the imported C
declaration. These functions are not printed into the generated compatibility
header because the declaration already exists.

## Module Selectors (SE-0491)

Use `ModuleName::` to disambiguate declarations from a specific imported module.
This solves cases where regular `ModuleName.member` syntax is shadowed by a type
or member with the same name.

```swift
import RocketEngine
import IonThruster

let primary = RocketEngine::ignite()
let backup = IonThruster::ignite()
```

Module selectors can also select extension members from a specific module:

```swift
let engine = Spacecraft.RocketEngine::Engine()
```

Use this syntax when:
- Two imported modules expose APIs with the same name.
- A module contains a top-level type with the same name as the module.
- Generated code or module interfaces need robust name qualification.
- A macro or attribute must be qualified by module.

Do not use module selectors on new declarations:

```swift
struct NASA::Scrubber { }  // Invalid
```

## Explicit Specialization (SE-0460)

Use `@specialized(where ...)` to ask the compiler to emit pre-specialized
implementations of a generic function for specific concrete types. This is
primarily for library authors and performance-critical generic APIs where the
caller cannot specialize the implementation itself.

```swift
extension Sequence where Element: BinaryInteger {
    @specialized(where Self == [Int])
    @specialized(where Self == [UInt32])
    func sumAsDouble() -> Double {
        reduce(0) { $0 + Double($1) }
    }
}
```

Key points:
- Replaces many uses of underscored `@_specialize`.
- The `where` clause must fully specify all generic placeholders.
- Dispatch uses exact type matches.
- The feature should not change semantics; it only changes performance.
- Each specialization can increase binary size. Add it only for measured hot paths.

## `@inline(always)` (SE-0496)

Use `@inline(always)` to require direct calls to be inlined when possible.

```swift
@inline(always)
func clamp(_ value: Int, min lowerBound: Int, max upperBound: Int) -> Int {
    min(max(value, lowerBound), upperBound)
}
```

Use sparingly:
- Appropriate for tiny, measured hot-path helpers.
- Avoid on non-final class methods; the compiler cannot guarantee inlining
  through dynamic dispatch.
- Avoid recursive cycles; they cannot be inlined completely.
- Do not use as a general-purpose performance decoration.

Migration:
- Prefer `@inline(always)` over underscored or implementation-detail spellings
  such as `@inline(__always)`.

## Function Definition Visibility (SE-0497)

Use `@export` to control whether a function emits a callable symbol and whether
its implementation is visible to clients for optimization.

```swift
@export(implementation)
public func smallBackDeployableHelper(_ value: Int) -> Int {
    value &* 31
}

@export(interface)
public func replaceableEntryPoint() {
    runCurrentImplementation()
}
```

Modes:
- `@export(implementation)` makes the function body available to clients for
  optimization. It subsumes many uses of `@_alwaysEmitIntoClient`.
- `@export(interface)` guarantees a callable symbol while hiding the function
  body from clients. Use it when the binary symbol must exist or implementation
  details must stay hidden.

Restrictions:
- Do not combine `@export` with `@inlinable`, `@usableFromInline`,
  `@_alwaysEmitIntoClient`, or `@_neverEmitIntoClient`.
- Generic functions are incompatible with `@export(interface)` in Embedded Swift,
  because generic specialization needs the implementation.
- Treat this as ABI and library-evolution surface area, not routine app code.

## Section Placement Control (SE-0492)

Use `@section` and `@used` on global or static stored variables for low-level
binary layout use cases such as linker sets, plugin metadata, test discovery,
or embedded startup contracts.

```swift
@section("__DATA,mysection")
@used
let pluginRecord: (version: Int, initializer: @convention(c) () -> Void) = (
    version: 1,
    initializer: { print("initialize plugin") }
)
```

Cross-platform section names should be conditionalized by object format:

```swift
#if objectFormat(ELF)
@section("mysection")
#elseif objectFormat(MachO)
@section("__DATA,mysection")
#endif
@used
let entry: Int = 42
```

Restrictions:
- Only global variables and static stored variables are supported.
- Local variables, instance stored properties, computed properties, observers,
  and generic contexts are not supported.
- Initializers must be constant expressions and statically initializable.
- Section names are passed through to the object format/linker; validate them
  for each platform you support.

## Improved Codable Error Debug Printing (SE-0489)

`EncodingError` and `DecodingError` now provide custom debug descriptions so
debug output is more readable.

```swift
do {
    _ = try JSONDecoder().decode(Person.self, from: data)
} catch {
    print(String(reflecting: error))
}
```

Guidance:
- Use this for human debugging output only.
- Do not parse `debugDescription`; switch on `EncodingError` or `DecodingError`
  cases and inspect associated values for program logic.
- Log error types explicitly if your logging pipeline relies on stable structure.

## `weak let` (SE-0481)

Immutable weak references are now supported. Use `weak let` when the weak
reference should never be reassigned, especially in `Sendable` classes and
`@Sendable` closures.

```swift
final class Session: Sendable {
    weak let user: User?

    init(user: User?) {
        self.user = user
    }
}

func makeHandler(user: User) -> @Sendable () -> Void {
    { [weak user] in
        user?.refresh()
    }
}
```

Key points:
- `weak let` must still be optional.
- A weak reference may observe `nil` after the object is destroyed, but the
  binding itself is immutable.
- Explicit weak captures are immutable; capture a separate `weak var` only when
  mutation is truly required.

## Clock Epochs (SE-0473)

`ContinuousClock` and `SuspendingClock` expose a `systemEpoch` property for the
clock's system-specific zero point.

```swift
let continuousClock = ContinuousClock()
let uptime = continuousClock.now - continuousClock.systemEpoch

let suspendingClock = SuspendingClock()
let activeTime = suspendingClock.now - suspendingClock.systemEpoch
```

Guidance:
- Use for comparing durations relative to the same system clock, including
  cross-process measurements on the same system.
- Do not serialize and compare these epochs across different systems.
- Platform definitions may vary.

## Additional Notes

- Swift 6.3 also includes an official Swift SDK for Android and build-system
  improvements, including Swift Build integration preview in SwiftPM.
- Swift Testing 6.3 includes warning issues, test cancellation, and image
  attachments; consult Swift Testing proposal references for detailed syntax.
