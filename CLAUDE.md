# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Git Commit Guidelines

**IMPORTANT: Follow these commit practices strictly**

### Atomic Commits with Conventional Commits Format

1. **Make atomic commits** - Each commit should contain a single, logical enhancement or change
2. **Use Conventional Commits format** - Follow the standard format:
   ```
   <type>(<scope>): <description>

   [optional body]

   [optional footer(s)]
   ```

3. **Common commit types:**
   - `feat:` - New feature
   - `fix:` - Bug fix
   - `docs:` - Documentation changes
   - `refactor:` - Code refactoring (no functional changes)
   - `test:` - Adding or updating tests
   - `chore:` - Maintenance tasks (dependencies, build config, etc.)
   - `perf:` - Performance improvements
   - `style:` - Code style changes (formatting, whitespace, etc.)

4. **Ask before moving to next task** - When transitioning from one feature/task to another, **ALWAYS ask the user if they want to commit the current changes** before proceeding with the next task.

5. **Examples:**
   ```
   feat(core): add support for custom event routers

   fix(serializer): resolve null reference in DomainId deserialization

   docs(readme): update handler registration examples for v8.3

   chore(build): fix Logo.png path in csproj file

   test(aggregate): add tests for event versioning
   ```

## Project Overview

Muflone is a C# library implementing CQRS (Command Query Responsibility Segregation) and Event Sourcing patterns, based on Jonathan Oliver's CommonDomain work from NEventStore. Current version: 8.5.0

**NuGet Package**: `Muflone`
**GitHub**: https://github.com/CQRS-Muflone/Muflone/
**Sample Usage**: https://github.com/CQRS-Muflone/CQRS-ES_testing_workshop

## Build, Test, and Development Commands

### Build
```bash
# Build entire solution
dotnet build src/muflone.sln

# Build specific project
dotnet build src/Muflone/Muflone.csproj
dotnet build src/Muflone.Tests/Muflone.Tests.csproj

# Build in Release mode (generates NuGet package due to GeneratePackageOnBuild=true)
dotnet build src/Muflone/Muflone.csproj -c Release
```

### Testing
```bash
# Run all tests
dotnet test src/muflone.sln

# Run tests with detailed output
dotnet test src/muflone.sln -v detailed

# Run specific test project
dotnet test src/Muflone.Tests/Muflone.Tests.csproj

# Run single test by filter
dotnet test --filter "FullyQualifiedName~AggregateRootTests"
dotnet test --filter "DisplayName~WhenCreatingAggregate"
```

### Package Management
```bash
# Restore dependencies
dotnet restore src/muflone.sln

# Pack NuGet package manually
dotnet pack src/Muflone/Muflone.csproj -c Release

# Clean build artifacts
dotnet clean src/muflone.sln
```

## Project Structure

```
src/
├── muflone.sln                    # Solution file
├── Muflone/                       # Main library project
│   ├── Muflone.csproj            # Targets net8.0, generates NuGet package on build
│   ├── Core/                      # Core DDD and Event Sourcing infrastructure
│   ├── Messages/                  # Commands, Events, and Handler abstractions
│   ├── Persistence/               # Repository and Serialization abstractions
│   └── Factories/                 # Handler factory interfaces
└── Muflone.Tests/                 # Test project using xUnit
    └── Muflone.Tests.csproj      # Test dependencies: xUnit, coverlet
```

## Core Architecture

### CQRS + Event Sourcing Pattern

Muflone implements a complete event-sourced CQRS architecture:

1. **Commands** (imperative) → trigger business logic → create **Events** (past-tense facts)
2. **Aggregates** enforce business rules and store state as a sequence of immutable events
3. **Events** are persisted to an event store and published to handlers
4. **Handlers** process commands and events asynchronously

### Key Abstractions

#### Domain Model (`src/Muflone/Core/`)
- **`AggregateRoot`** - Base class for aggregates with event sourcing capabilities
  - Manages event routing via `IRouteEvents` (ConventionEventRouter or RegistrationEventRouter)
  - `RaiseEvent(object)` - Applies event to aggregate and tracks as uncommitted
  - `ApplyEvent(object)` - Replays persisted event during aggregate reconstruction
  - Version tracking for optimistic concurrency control
  - Optional snapshot support via `IMemento`

- **`Entity`** - Base class for domain entities with identity-based equality
- **`ValueObject`** - Base class for value objects with structural equality
- **`DomainId`** - Type-safe aggregate identity (inherit and implement strongly-typed IDs)

#### Event Routing Strategies
- **`ConventionEventRouter`** - Auto-discovers `Apply(EventType)` methods via reflection
  - Low boilerplate: just implement `void Apply(MyEvent e)` methods
  - Throws `HandlerForDomainEventNotFoundException` if handler missing (configurable)

- **`RegistrationEventRouter`** - Explicit handler registration via `Register<T>(Action<T>)`
  - High control: manually wire event types to handlers

#### Message Hierarchy (`src/Muflone/Messages/`)
```
IMessage
├── ICommand → Command (abstract)
└── IEvent → Event (abstract)
    ├── IDomainEvent → DomainEvent (abstract)
    └── IIntegrationEvent → IntegrationEvent (abstract)
```

- **Commands** - Point-to-point, imperative instructions (e.g., `CreateCartCommand`)
  - Properties: `AggregateId`, `MessageId`, `Who` (Account), `When`, `UserProperties`

- **Domain Events** - Business facts within same bounded context (e.g., `CartCreatedEvent`)
  - Applied to aggregates via `Apply()` methods
  - Persisted to event store
  - Published to domain event handlers

- **Integration Events** - Cross-service communication for eventual consistency
  - Published via `IEventBus` to other microservices

#### Handler Registration (v8.3+ Auto-Consumer Feature)

**New simplified registration** (no manual consumer wrapping needed):
```csharp
builder.Services.AddCommandHandler<CreateCartHandler>();
builder.Services.AddDomainEventHandler<CartCreatedHandler>();
builder.Services.AddIntegrationEventHandler<ProductCreatedHandler>();
builder.Services.AddGenericHandler<MyCustomHandler>();
```

**How it works**:
- `MessageHandlerExtension.cs` tracks handler types in static list
- `MessageSubscriberBase` discovers `IMessageHandlerAsync<T>` interfaces via reflection
- Creates subscriptions automatically with DI resolution
- Eliminates manual consumer boilerplate (old `AddMufloneRabbitMQConsumers` approach still supported)

#### Handler Base Classes
- `CommandHandlerAsync<TCommand>` - Process commands, modify aggregates
- `DomainEventHandlerAsync<TEvent>` - React to domain events (same bounded context)
- `IntegrationEventHandlerAsync<TEvent>` - Handle events from other services

All implement `IMessageHandlerAsync<TMessage>` with single method:
```csharp
Task HandleAsync(TMessage message, CancellationToken cancellationToken)
```

### Persistence Layer (`src/Muflone/Persistence/`)

- **`IRepository`** - Aggregate storage abstraction
  - `GetByIdAsync<TAggregate>(IDomainId id, CancellationToken)` - Load aggregate
  - `GetByIdAsync<TAggregate>(IDomainId id, long version, CancellationToken)` - Load specific version
  - `SaveAsync(IAggregate aggregate, Guid commitId, CancellationToken)` - Persist uncommitted events

- **`ISerializer`** - Message serialization (default: JSON with type information via Newtonsoft.Json)
  - `DeserializeAsync<T>(string, CancellationToken)`
  - `SerializeAsync<T>(T, CancellationToken)`

- **`IServiceBus`** - Send commands to transport
- **`IEventBus`** - Publish events to subscribers

### Concurrency Control

- **`IDetectConflicts`** / **`ConflictDetector`** - Optimistic concurrency via event conflict detection
  - Register conflict predicates: `Register<TUncommitted, TCommitted>(delegate)`
  - Default: events of same type conflict
  - Custom logic: register specific event pair handlers
  - Throws `ConflictingCommandException` on conflict

### Transport Abstraction

Muflone is transport-agnostic via:
- **`ITransporterConnectionFactory`** - Marker interface for connection factories
- **`IConsumer`** - Lifecycle interface (`StartAsync`, `StopAsync`)
- **`MessageSubscriberBase<TChannel>`** - Abstract base for transport-specific implementations
  - Override: `InitChannelAsync`, `InitSubscriptionAsync`, `StopChannelAsync`
  - Example: Muflone.RabbitMQ extends this for RabbitMQ transport

## Development Workflow

### Data Flow: Command → Event → Handler

```
1. Command sent via IServiceBus
2. CommandHandler receives command
3. Handler loads/creates aggregate via IRepository
4. Aggregate.RaiseEvent() applies event and tracks as uncommitted
5. IRepository.SaveAsync() persists events to event store
6. Events published to DomainEventHandlers (same service) and IntegrationEventHandlers (other services)
7. Handlers react asynchronously
```

### Creating New Aggregates

```csharp
public class Cart : AggregateRoot
{
    public Cart(CartId id)
    {
        // Register event handlers (ConventionEventRouter auto-discovers these)
        Register<CartCreated>(Apply);
        Register<ItemAdded>(Apply);
    }

    public static Cart Create(CartId id, UserId userId)
    {
        var cart = new Cart(id);
        cart.RaiseEvent(new CartCreated(id, userId)); // Applies + tracks
        return cart;
    }

    void Apply(CartCreated e) { /* Update aggregate state */ }
    void Apply(ItemAdded e) { /* Update aggregate state */ }
}
```

### Creating Strongly-Typed IDs

```csharp
public sealed record CartId(Guid Value) : DomainId(Value.ToString())
{
    public static CartId New() => new CartId(Guid.NewGuid());
}
```

### Implementing Handlers

```csharp
public class CreateCartHandler : CommandHandlerAsync<CreateCartCommand>
{
    private readonly IRepository _repository;

    public CreateCartHandler(IRepository repository, /* other deps */)
    {
        _repository = repository;
    }

    public override async Task HandleAsync(CreateCartCommand command, CancellationToken cancellationToken)
    {
        var cart = Cart.Create(command.CartId, command.UserId);
        await _repository.SaveAsync(cart, command.MessageId, cancellationToken);
    }
}
```

## Testing

- **Framework**: xUnit
- **Test Organization**: Tests mirror the source structure in `Muflone.Tests/`
- **Coverage**: coverlet.collector configured for code coverage

### Example Test Structure
```
Muflone.Tests/
├── Core/
│   ├── AggregateRootTests.cs
│   ├── EntityTests.cs
│   ├── ValueObjectTests.cs
│   └── DomainIdTests.cs
├── Messages/
│   └── EventTests.cs
├── Persistence/
│   └── SerializerTests.cs
└── Integration/
    └── CommandEventFlowTests.cs
```

## Important Dependencies

- **Microsoft.Extensions.Hosting.Abstractions** (8.0.1) - IHostedService integration
- **Microsoft.Extensions.Logging** (8.0.0) - Logging abstraction
- **Newtonsoft.Json** (13.0.3) - JSON serialization with type information
- **NewId** (4.0.1) - Distributed unique ID generation

## Version History Notes

### v8.5.0 (Current)
- Improved DomainId deserialization (no longer requires property named `aggregateId`)

### v8.3.0
- Auto-consumer registration feature
- Register handlers directly without manual consumer wrapping
- `AddCommandHandler<T>()`, `AddDomainEventHandler<T>()`, `AddIntegrationEventHandler<T>()`

## Key Design Decisions

1. **Nullable Reference Types Enabled** - All new code must handle nullability correctly
2. **Async-First** - All I/O operations use async/await with CancellationToken support
3. **Type Safety** - Strong typing via generics and compile-time checking
4. **Immutable Events** - Events are immutable records of what happened
5. **Version Tracking** - Every event increments aggregate version for optimistic concurrency
6. **Audit Trail** - Events include `Who` (Account) and `When` (timestamp) metadata
7. **Transport Agnostic** - Core library doesn't depend on specific message transport (RabbitMQ, Azure Service Bus, etc.)

## Common Patterns

### Deserializing DomainId from Messages
Since v8.5.0, DomainId can be deserialized without requiring property name `aggregateId`:
```csharp
public sealed record CartId(Guid Value) : DomainId(Value.ToString());

// This works now (property name doesn't matter)
public class MyEvent : DomainEvent
{
    public CartId CartIdentifier { get; init; } // ← Can be any name
}
```

### Custom Header Management
```csharp
var headers = new EventHeaders();
headers.Set("CorrelationId", correlationId);
headers.Set("CustomKey", customValue);
var customValue = headers.Get("CustomKey");
```

### Snapshot Support (Performance Optimization)
```csharp
public class LargeAggregate : AggregateRoot
{
    public override IMemento GetSnapshot()
    {
        return new MySnapshot(Id.ToString(), Version, /* state */);
    }
}
```
