<!--
Sync Impact Report:
- Version: 1.1.0 (minor amendment)
- Changes from 1.0.0:
  * Added "Project Mission" section clarifying translation objective
  * Constitution now explicitly governs both C# and PHP implementations
  * Reference to MUFLONE-PHP-TRANSLATION-PROJECT.md for PHP-specific guidance
- Principles (unchanged):
  1. Event Sourcing Integrity
  2. CQRS Separation
  3. Test-First Development
  4. Transport Agnosticism
  5. Backward Compatibility
- Templates status:
  ✅ plan-template.md - no changes required
  ✅ spec-template.md - no changes required
  ✅ tasks-template.md - no changes required
- Follow-up: None required
-->

# Muflone Constitution

## Project Mission

**Primary Objective**: Translate the Muflone CQRS/Event Sourcing library from C# (.NET Core 8) to PHP 8.4.

This constitution governs both:
1. **Maintenance and enhancement** of the existing C# library
2. **Translation and development** of the PHP implementation

All principles below apply to both implementations, with language-specific adaptations where necessary (documented in `MUFLONE-PHP-TRANSLATION-PROJECT.md` for PHP).

## Core Principles

### I. Event Sourcing Integrity

**MUST**:
- All state changes MUST flow through events via `RaiseEvent()`
- Events MUST be immutable once created
- Events MUST use past-tense naming (e.g., `CartCreated`, not `CreateCart`)
- Aggregates MUST reconstruct state by replaying events through `Apply()` methods
- Event schemas MUST NOT be modified after release (create new event versions instead)

**Rationale**: Event sourcing provides complete audit trail and temporal queries. Breaking event immutability destroys historical accuracy and breaks aggregate reconstruction.

### II. CQRS Separation

**MUST**:
- Commands MUST be imperative (e.g., `CreateCartCommand`)
- Commands MUST target a single aggregate
- Domain Events MUST remain within bounded context
- Integration Events MUST be used for cross-service communication
- Queries MUST NOT modify state (read models separate from write models)

**Rationale**: Clear separation between commands (write) and queries (read) enables independent scaling, optimization, and evolution of each side.

### III. Test-First Development (NON-NEGOTIABLE)

**MUST**:
- Unit tests MUST be written before implementation for all new features
- Tests MUST cover: aggregate behavior, event application, command handling, conflict detection
- Tests MUST verify idempotency where applicable
- Integration tests MUST cover end-to-end command→event→handler flows
- All tests MUST pass before merge

**SHOULD**:
- Test coverage SHOULD be >= 80% for core domain logic
- Property-based testing SHOULD be used for value objects and entities

**Rationale**: CQRS/ES systems are complex with event ordering, versioning, and concurrency concerns. Tests are the only way to ensure correctness.

### IV. Transport Agnosticism

**MUST**:
- Core library MUST NOT depend on specific transport implementations (RabbitMQ, Azure Service Bus, etc.)
- All transport-specific code MUST live in separate packages (e.g., Muflone.RabbitMQ)
- Message contracts (`ICommand`, `IEvent`) MUST remain transport-independent
- Serialization MUST be pluggable via `ISerializer` interface

**SHOULD**:
- Transport packages SHOULD follow naming convention: `Muflone.<TransportName>`
- Examples SHOULD demonstrate multiple transport implementations

**Rationale**: Transport flexibility enables adoption in diverse environments and prevents vendor lock-in.

### V. Backward Compatibility

**MUST**:
- Public APIs MUST follow semantic versioning (MAJOR.MINOR.PATCH)
- Breaking changes MUST increment MAJOR version
- Event schema changes MUST be additive only OR versioned as new event types
- Deprecated features MUST be marked with `[Obsolete]` attribute for one MAJOR version before removal
- CHANGELOG MUST document all breaking changes with migration guide

**SHOULD**:
- New features SHOULD increment MINOR version
- Bug fixes SHOULD increment PATCH version

**Rationale**: Muflone is a library used by production systems. Breaking existing consumers without clear migration paths is unacceptable.

## Quality Standards

### Code Quality

**MUST**:
- Nullable reference types MUST be enabled (`<Nullable>enable</Nullable>`)
- All public APIs MUST have XML documentation comments
- All async methods MUST accept `CancellationToken`
- All I/O operations MUST be async (no blocking calls)

**SHOULD**:
- Code SHOULD follow C# naming conventions (PascalCase for public members, camelCase for private)
- Cyclomatic complexity SHOULD be <= 10 per method
- Classes SHOULD follow Single Responsibility Principle

### Performance

**MUST**:
- Event replay MUST be optimized (support snapshots via `IMemento`)
- Serialization MUST handle large event streams efficiently
- Memory allocations SHOULD be minimized in hot paths

**SHOULD**:
- Benchmarks SHOULD exist for critical paths (event application, serialization)
- Performance regressions SHOULD be caught in CI

## Development Workflow

### Feature Development Process

1. **Specification**: Use `/speckit.specify` to document feature requirements
2. **Planning**: Use `/speckit.plan` to design architecture and data model
3. **Task Breakdown**: Use `/speckit.tasks` to create actionable task list
4. **Implementation**: Use `/speckit.implement` with test-first approach
5. **Review**: Verify constitution compliance before merge

### Commit Standards

**MUST**:
- Follow Conventional Commits format: `<type>(<scope>): <description>`
- Commit types: `feat`, `fix`, `docs`, `refactor`, `test`, `chore`, `perf`, `style`
- Each commit MUST be atomic (single logical change)
- Breaking changes MUST include `BREAKING CHANGE:` in commit footer

### Documentation Requirements

**MUST**:
- New features MUST update CLAUDE.md with usage examples
- Breaking changes MUST include migration guide in docs
- Real-world examples MUST be added to MUFLONE_ANALYSIS.md when significant patterns emerge

**SHOULD**:
- Architecture decisions SHOULD be documented in GEMINI.md
- Complex patterns SHOULD include sequence diagrams

## Governance

### Amendment Process

1. Proposed amendments MUST be discussed with project maintainers
2. Constitution changes MUST increment version according to semantic versioning
3. Major principle changes require approval from core team
4. All PRs MUST verify compliance with current constitution

### Compliance Verification

**Every feature implementation MUST**:
- Pass `/speckit.analyze` with zero CRITICAL issues
- Have test coverage meeting standards
- Follow commit conventions
- Update relevant documentation

**Constitution authority**: This constitution is non-negotiable during feature implementation. If a principle conflicts with a requirement, the requirement must be adjusted or the constitution must be amended through proper governance process.

### Runtime Guidance

See `CLAUDE.md` for AI assistant guidance on:
- Build and test commands
- Architecture overview
- Common development patterns
- Handler registration approaches

## Resources
- Muflone GitHub repository: [Muflone](https://github.com/CQRS-Muflone/Muflone)

**Version**: 1.1.0 | **Ratified**: 2025-01-02 | **Last Amended**: 2025-01-02
