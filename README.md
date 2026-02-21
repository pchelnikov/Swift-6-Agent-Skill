# Swift Modern Patterns Skill

Ensure your AI coding tool generates and reviews Swift code using the latest language features from Swift 6.0 through 6.2 — concurrency, typed throws, noncopyable types, observation, testing, and more.

Built on the [Agent Skills open format](https://agentskills.io/home). Every recommendation is grounded in a Swift Evolution proposal.

## Who This Is For

- Teams migrating from Swift 5.x to Swift 6.x who need correct, version-aware guidance
- Developers generating new Swift code and wanting modern idioms applied automatically
- Anyone reviewing or refactoring Swift code for outdated patterns (deprecated APIs, concurrency warnings, legacy error handling)
- Projects that need their AI agent to know what's available in Swift 6.0, 6.1, and 6.2

## How to Use This Skill

### Option A: Using skills.sh (recommended)

Install this skill with a single command:

```
npx skills add https://github.com/pchelnikov/swift-6-agent-skill --skill swift-6-agent-skill
```

Then use the skill in your AI agent, for example:

> Use the swift modern patterns skill and review this file for outdated Swift patterns and concurrency issues

### Option B: Claude.ai

Upload the `swift-6-agent-skill/` folder as a zip file through **Settings > Features** (requires Pro, Max, Team, or Enterprise plan with code execution enabled).

### Option C: Claude API

Upload via the `/v1/skills` endpoint and reference by `skill_id` in your API calls. See [Using Skills with the Claude API](https://platform.claude.com/docs/en/build-with-claude/skills-guide).

### Option D: Manual Install

1. **Clone** this repository.
2. **Install or symlink** the `swift-6-agent-skill/` folder following your tool's official skills installation docs (see links below).
3. **Use your AI tool** as usual — the skill triggers automatically on any Swift code generation, review, or refactoring task.

#### Where to Save Skills

Follow your tool's official documentation:

- **Codex:** [Where to save skills](https://developers.openai.com/codex/skills/#where-to-save-skills)
- **Claude:** [Using Skills](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#using-skills)
- **Cursor:** [Enabling Skills](https://cursor.com/docs/context/skills#enabling-skills)

**How to verify:** Your agent should reference the workflow decision tree and checklists in `swift-6-agent-skill/SKILL.md` and jump into the relevant reference file for your task.

## What This Skill Offers

This skill gives your AI coding tool version-aware Swift language guidance. It can:

### Generate Modern Swift Code

- Apply current concurrency patterns: default MainActor isolation (SE-0466), `Task.immediate` (SE-0472), `isolated deinit` (SE-0371)
- Use typed throws for constrained error domains (SE-0413)
- Use `InlineArray` for fixed-size collections (SE-0453), `weak let` for Sendable types (SE-0481)
- Apply modern syntax: trailing commas, raw identifiers, string interpolation defaults

### Review and Refactor Existing Code

- Detect outdated patterns with a 16-entry old → new quick reference table
- Systematically audit code with a categorized review checklist (concurrency, errors, imports, testing)
- Triage common concurrency compiler errors with solution routing

### Guide Swift 5.x → 6.x Migration

- Step-by-step migration patterns with before/after code examples
- Concurrency safety: global variables, actor isolation, Sendable conformance
- Import visibility cleanup, test suite upgrades, and language mode changes

### Improve Swift Testing

- Prefer Swift Testing over XCTest for new targets
- Use exit tests for `precondition`/`fatalError` paths (ST-0008)
- Use test scoping traits for concurrency-safe shared configuration (ST-0007)
- Use raw identifiers for human-readable test names (SE-0451)

## What Makes This Skill Different

**SE-Proposal Grounded:** Every single recommendation traces to a Swift Evolution proposal. No opinions, no folklore — just what the language team shipped.

**Version-Aware:** Every feature is tagged with the Swift version that introduced it. The agent won't suggest Swift 6.2 features to a project targeting Swift 6.0.

**Non-Opinionated:** No architecture enforcement (MVVM, TCA, VIPER). No formatting or linting rules. We cover language features; how you structure your app is your decision.

**Decision-Tree Driven:** The SKILL.md uses a workflow decision tree (generate → review → migrate → debug → explain) with inline routing to reference files — modeled on proven patterns from production Agent Skills.

**Practical & Concise:** Treats the agent as capable. Provides checklists and quick-reference tables, not tutorials.

## Skill Structure

```
swift-6-agent-skill/
├── SKILL.md                  # Decision tree, review checklist, quick reference, guidelines
└── references/
    ├── SWIFT_6_0.md          # Concurrency defaults, Mutex/Atomics, typed throws,
    │                         #   @DebugDescription, noncopyable upgrades, AsyncSequence,
    │                         #   RangeSet, Int128, BitwiseCopyable
    ├── SWIFT_6_1.md          # Trailing commas, metatype key paths, TaskGroup inference,
    │                         #   nonisolated types, import visibility, diagnostic groups
    ├── SWIFT_6_2.md          # Default MainActor isolation, raw identifiers, InlineArray
    │                         #   + [N of T] sugar, immediate tasks, weak let, Observations,
    │                         #   Span/MutableSpan, method key paths, feature adoption tooling
    ├── SWIFT_6_2_3.md        # Extensible enums for non-resilient modules (SE-0487)
    └── MIGRATION.md          # Old → new pattern mappings with before/after code examples
```

## Token Budget

The skill uses progressive disclosure to minimize context window usage:

| Load Level | When | ~Tokens |
|------------|------|---------|
| Metadata only | Every conversation | ~130 |
| SKILL.md triggered | Swift task detected | ~5,000 |
| + One reference file | Version-specific detail needed | ~6,500–8,000 |
| All files loaded | Worst case (rare) | ~15,000 |

## Sources

Feature coverage is based on these references:

- [Announcing Swift 6](https://www.swift.org/blog/announcing-swift-6/) — Official Swift.org blog
- [Swift 6.1 Released](https://www.swift.org/blog/swift-6.1-released/) — Official Swift.org blog
- [Swift 6.2 Released](https://www.swift.org/blog/swift-6.2-released/) — Official Swift.org blog
- [What's new in Swift 6.0](https://www.hackingwithswift.com/articles/269/whats-new-in-swift-6) by Paul Hudson
- [What's new in Swift 6.1](https://www.hackingwithswift.com/articles/276/whats-new-in-swift-6-1) by Paul Hudson
- [What's new in Swift 6.2](https://www.hackingwithswift.com/articles/277/whats-new-in-swift-6-2) by Paul Hudson
- [Swift Evolution Proposals](https://github.com/swiftlang/swift-evolution) — all implemented proposals for Swift 6.0, 6.1, 6.2, and 6.2.3

## Contributing

Contributions are welcome! When submitting changes:

- Every recommendation must reference a Swift Evolution proposal (SE-####) or Swift Testing proposal (ST-####)
- Code examples should be minimal and self-contained
- Tag every feature with the Swift version that introduced it
- Do not add architecture opinions, formatting rules, or linting preferences

## License

This skill is open-source and available under the MIT License. See [LICENSE](LICENSE) for details.
