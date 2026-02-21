# Swift 6.2.3 Features Reference

Point release with one significant language change.

## Extensible Enums (SE-0487)

Public enums in non-resilient Swift libraries (packages) can now be extensible, allowing new cases to be added without a SemVer-major breaking change.

### The Problem

In Swift packages (non-resilient modules), adding a new case to a public enum has always been a source-breaking change for consumers, because their `switch` statements become non-exhaustive. This made enums impractical for evolving public APIs — particularly error types.

### The Solution

Public enums in non-resilient modules are now extensible by default (matching the behavior of resilient/ABI-stable libraries). Consumers must handle unknown future cases with `@unknown default`.

Use `@frozen` to explicitly opt out and declare that the enum's case list is fixed.

### Syntax

```swift
// Library code:

// Extensible (default) — new cases can be added in future versions
public enum NetworkError: Error {
    case timeout
    case connectionLost
    case invalidResponse
    // Future version can add: case rateLimited
}

// Frozen — case list is fixed, adding a case is a breaking change
@frozen
public enum Direction {
    case north, south, east, west
}
```

```swift
// Consumer code:

// Must handle future cases for extensible enums
switch error {
case .timeout:
    retry()
case .connectionLost:
    reconnect()
case .invalidResponse:
    reportBug()
@unknown default:
    handleGenericError(error)  // Handles future cases safely
}

// Frozen enums can be exhaustively matched (no @unknown default needed)
switch direction {
case .north: goNorth()
case .south: goSouth()
case .east: goEast()
case .west: goWest()
}
```

### When to Use @frozen

- The enum represents a mathematically complete set (compass directions, binary states).
- The enum is part of a stable, finalized API where adding cases would be semantically wrong.
- Performance-critical code where the compiler can optimize exhaustive switches.

### When NOT to Use @frozen

- Error enums — new error conditions are almost always possible.
- Status/state enums in evolving APIs.
- Any enum where you might conceivably add a case in a future version.

### Migration Notes

- **Existing packages**: If you have public enums and want to preserve the current (non-extensible) behavior, add `@frozen` to them before consumers start depending on exhaustive matching.
- **New packages**: Leave enums extensible by default unless you have a strong reason to freeze them.
- **Consumers**: Add `@unknown default` to `switch` statements over public enums from dependencies to future-proof your code.
