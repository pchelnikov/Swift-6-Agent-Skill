# Swift 6.4 Features Reference

Released: September 2026. Compiler: Swift 6.4+.

Use the [Swift Evolution 6.4 implemented filter](https://www.swift.org/swift-evolution/#?version=6.4) as the release inventory. It lists the 26 SE proposals below. Check the linked proposal before adopting a specialized API: some features concern SwiftPM, the runtime, or library authors rather than ordinary app code. Swift Testing proposals have their own ST numbering and are covered separately at the end.

## Concurrency and observation

- [SE-0530](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0530-async-result-support.md) adds an async `Result.init(catching:)`. Use `let result = await Result { try await work() }` when an async error should be captured as a value.
- [SE-0528](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0528-noncopyable-continuation.md) adds noncopyable `Continuation` and `withContinuation(of:throwing:_:)`. A `resume` consumes the continuation, so double resume is rejected by the compiler; dropping it without resuming traps. Keep `CheckedContinuation` when multiple escaping callbacks must share the completion path.
- [SE-0523](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0523-hashable-unownedtask-executor.md) makes `UnownedTaskExecutor` `Hashable`, useful when tracking or deduplicating executor identities.
- [SE-0520](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0520-discardableresult-task-initializers.md) makes the result of `Task` initializers discardable when the task operation returns `Void`. Retain the task handle when cancellation or awaiting its result matters.
- [SE-0506](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0506-advanced-observation-tracking.md) adds configurable `withObservationTracking(options:_:onChange:)` and continuous observation tracking. Choose these when synchronous, fine-grained change delivery is needed; `Observations` remains useful for async streams.
- [SE-0504](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0504-task-cancellation-shields.md) adds `withTaskCancellationShield` for cleanup that must finish despite cancellation. Shield only the necessary work; it does not cancel-proof the entire enclosing operation.
- [SE-0493](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0493-defer-async.md) permits `await` in `defer` inside an async context. Deferred async work completes before scope exit, including throwing or early-return paths. Combine it with a cancellation shield when cleanup must ignore cancellation.

## Ownership, collections, and memory

- [SE-0527](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0527-rigidarray-uniquearray.md) adds `UniqueArray` for dynamically sized, uniquely owned storage of noncopyable elements. Use it when `Array` copy-on-write or element copyability is the constraint.
- [SE-0525](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0525-rawspan-safe-loading-api.md) adds safe loading APIs for `RawSpan` and related views. Prefer these over unsafe-annotated raw-memory loads when their layout and bounds requirements fit.
- [SE-0524](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0524-span-temporary-allocation.md) adds `withTemporaryAllocation` with output span types for temporary, initialized scratch storage.
- [SE-0519](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0519-ref-mutableref-types.md) adds `Ref` and `MutableRef`, storable references that retain shared or exclusive access to a value without an unsafe pointer.
- [SE-0517](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0517-uniquebox.md) adds `UniqueBox` for uniquely owned heap storage, including noncopyable values.
- [SE-0516](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0516-borrowing-sequence.md) adds `Iterable` for iteration that can borrow noncopyable elements instead of requiring `Sequence`-style copies.
- [SE-0514](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0514-hashable-conformance-for-dictionarykeys-collectionofone-emptycollection.md) adds `Hashable` conformances to `Dictionary.Keys`, `CollectionOfOne`, and `EmptyCollection` when their elements support hashing.
- [SE-0499](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0499-support-non-copyable-simple-protocols.md) extends simple standard-library protocols to noncopyable and nonescapable conforming types.
- [SE-0494](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0494-add-is-identical-methods.md) adds `isTriviallyIdentical(to:)` for fast identity checks on supported concrete types. It is not a replacement for semantic equality.

## Language and diagnostics

- [SE-0522](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0522-source-warning-control.md) adds `@diagnose(GroupName, as: warning|error|ignored)` to set warning behavior in a declaration's lexical scope. Use a real diagnostic group and a narrow scope; supply a reason when suppressing a warning.
- [SE-0521](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0521-improved-optional-opaque-and-any.md) permits `some P?` and `any P?` in place of `(some P)?` and `(any P)?`.
- [SE-0518](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0518-tilde-sendable.md) adds `~Sendable` on a struct, enum, or class declaration to explicitly suppress inferred `Sendable` conformance. It does not make unsafe sharing safe; a subclass can independently establish `Sendable` conformance.
- [SE-0508](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0508-array-expression-trailing-closures.md) allows a trailing closure after an array type in an expression when an applicable initializer accepts it.
- [SE-0507](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0507-borrow-accessors.md) adds `borrow` and `mutate` property accessors for access without copying. `mutate` requires `borrow`; accessors must yield storage with a valid lifetime. Use them for ownership-sensitive abstractions, not routine stored properties.
- [SE-0503](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0503-suppressed-associated-types.md) allows suppression of default `Copyable` and `Escapable` requirements on associated types with defaults. The proposal still documents an experimental feature flag; verify toolchain support before recommending it in production code.
- [SE-0502](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0502-exclude-private-from-memberwise-init.md) excludes a privately scoped, initialized stored property from a struct's synthesized memberwise initializer. Review initializer call sites when changing a property's access level or default value.

## Tooling and runtime

- [SE-0511](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0511-swiftpm-add-target-plugin.md) adds a SwiftPM command for adding a target plugin to a package.
- [SE-0509](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0509-swift-sboms-via-swiftpm.md) adds SwiftPM software bill of materials generation in SPDX or CycloneDX format for dependency inventory workflows.
- [SE-0498](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0498-runtime-demangle.md) exposes demangling through the Runtime module for tooling that needs to render Swift symbol names.

## Swift Testing in the 6.4 release

These ST proposals appear in the [Swift 6.4 release announcement](https://www.swift.org/blog/swift-6.4-released/), rather than the dashboard's SE-only implemented filter:

- [ST-0021](https://github.com/swiftlang/swift-evolution/blob/main/proposals/testing/0021-targeted-interoperability-swift-testing-and-xctest.md): targeted interoperability lets migration helpers use `XCTAssert` in Swift Testing tests and `#expect` in XCTest tests.
- [ST-0022](https://github.com/swiftlang/swift-evolution/blob/main/proposals/testing/0022-customtestreflectable.md): `CustomTestReflectable` customizes values shown in failed expectations.
- [ST-0023](https://github.com/swiftlang/swift-evolution/blob/main/proposals/testing/0023-attachments-transferable.md): attachments can use `Transferable` on Apple platforms.
- [ST-0024](https://github.com/swiftlang/swift-evolution/blob/main/proposals/testing/0024-per-test-case-repetitions.md): `swift test` can repeat individual test cases with repetition options.
