# Muflone PHP Translation Project

## Executive Summary

This document describes the comprehensive project to translate the **Muflone** CQRS (Command Query Responsibility Segregation) and Event Sourcing library from C# (.NET Core 8) to PHP 8.4.

**Original Repository**: https://github.com/CQRS-Muflone/Muflone
**Target**: Vanilla PHP 8.4 library (framework-agnostic)
**Approach**: Faithful translation with PHP idioms and best practices

---

## Project Goals

### Primary Objectives

1. **Complete Functional Parity**: Replicate all features of the C# library
2. **PHP Best Practices**: Adapt to PHP idioms and modern language features
3. **Framework Independence**: Pure PHP without Symfony/Laravel dependencies
4. **Production Ready**: Well-tested, documented, and easy to use
5. **Type Safety**: Leverage PHP 8.4 type system with PHPStan level 8 compliance

### Success Criteria

- ✅ All C# components translated to PHP
- ✅ Test coverage > 80% (ideally > 90%)
- ✅ PHPStan level 8 passes with zero errors
- ✅ Comprehensive documentation with examples
- ✅ Docker-based development environment
- ✅ PSR-1, PSR-4, PSR-11, PSR-12 compliant

---

## Project Scope

### What is Muflone?

Muflone is a CQRS and Event Sourcing library based on Jonathan Oliver's CommonDomain work from NEventStore. It provides:

- **Aggregate Root** pattern for domain modeling
- **Command/Query Separation** for scalable architectures
- **Event Sourcing** for complete audit trails
- **Domain Events** and **Integration Events** for system decoupling
- **Repository Pattern** for aggregate persistence
- **Optimistic Concurrency** control

### Translation Philosophy

**Preserve Intent, Adapt Implementation**

- Maintain the same architectural patterns
- Keep the same public API concepts
- Adapt language-specific features (generics → templates, async → sync, LINQ → array functions)
- Follow PHP conventions while respecting CQRS/ES principles
- Document all significant translation decisions

---

## Technical Architecture

### Target Directory Structure

```
muflone-php/
├── src/
│   ├── Core/
│   │   ├── Abstractions/       # Core interfaces
│   │   ├── Domain/              # Aggregate, Entity, ValueObject
│   │   ├── Commands/            # Command base classes and interfaces
│   │   ├── Events/              # Event base classes and interfaces
│   │   └── Exceptions/          # Custom exceptions
│   ├── Messaging/
│   │   ├── CommandHandling/     # Command handler and dispatcher
│   │   ├── EventHandling/       # Event handler and dispatcher
│   │   └── Transport/           # Message transport abstraction
│   ├── Persistence/
│   │   ├── EventStore/          # Event store abstraction
│   │   └── Repository/          # Repository pattern implementation
│   └── Infrastructure/
│       ├── Serialization/       # JSON serialization utilities
│       └── DependencyInjection/ # PSR-11 DI container
├── tests/
│   ├── Unit/                    # Unit tests
│   ├── Integration/             # Integration tests
│   ├── Functional/              # End-to-end tests
│   └── Fixtures/                # Test data
├── examples/                    # Working code examples
├── docs/                        # Architecture and API documentation
└── docker/                      # Docker configuration
```

### PHP Standards Compliance

- **PSR-1**: Basic Coding Standard
- **PSR-4**: Autoloading
- **PSR-11**: Container Interface (Dependency Injection)
- **PSR-12**: Extended Coding Style
- **PSR-3**: Logger Interface (optional)

---

## Key Translation Patterns

### 1. C# Generics → PHP Templates with DocBlocks

**C#:**
```csharp
public class Handler<TCommand> where TCommand : ICommand
{
    public async Task HandleAsync(TCommand command) { }
}
```

**PHP 8.4:**
```php
/**
 * @template TCommand of ICommand
 */
class Handler
{
    /**
     * @param TCommand $command
     */
    public function handle(ICommand $command): void
    {
        // implementation
    }
}
```

### 2. C# Async/Await → PHP Synchronous Methods

**C#:**
```csharp
public async Task<Result> HandleAsync(Command cmd)
{
    var result = await _repository.SaveAsync(aggregate);
    return result;
}
```

**PHP:**
```php
public function handle(Command $cmd): Result
{
    // Synchronous implementation with clear naming
    $result = $this->repository->save($aggregate);
    return $result;
}
```

**Rationale**: PHP lacks native async/await. Keep methods synchronous and well-documented. If async behavior is needed, document ReactPHP or Swoole as optional extensions.

### 3. C# LINQ → PHP Array Functions / Collections

**C#:**
```csharp
var result = events
    .Where(e => e.Type == "Created")
    .Select(e => e.Data)
    .ToList();
```

**PHP:**
```php
// Native approach
$filtered = array_filter($events, fn($e) => $e->getType() === 'Created');
$result = array_map(fn($e) => $e->getData(), $filtered);

// Or with custom Collection class
$result = Collection::from($events)
    ->filter(fn($e) => $e->getType() === 'Created')
    ->map(fn($e) => $e->getData())
    ->toArray();
```

### 4. C# Attributes → PHP 8 Attributes

**C#:**
```csharp
[CommandHandler]
public class CreateCartHandler : ICommandHandler
{
    // ...
}
```

**PHP:**
```php
#[CommandHandler]
class CreateCartHandler implements CommandHandler
{
    // ...
}
```

### 5. C# Properties → PHP 8.4 Asymmetric Visibility

**C#:**
```csharp
public string Name { get; private set; }
```

**PHP 8.4:**
```php
public private(set) string $name;
```

### 6. C# Extension Methods → PHP Traits / Helper Classes

**C#:**
```csharp
public static class Extensions
{
    public static void DoSomething(this MyClass obj) { }
}
```

**PHP:**
```php
// Option 1: Trait
trait MyClassBehavior
{
    public function doSomething(): void { }
}

// Option 2: Static Helper
class MyClassHelper
{
    public static function doSomething(MyClass $obj): void { }
}
```

---

## Core Components to Implement

### 1. Aggregate Root

```php
<?php

declare(strict_types=1);

namespace Muflone\Core\Domain;

use Muflone\Core\Events\DomainEvent;

/**
 * Base class for all Domain Aggregate Roots
 *
 * An Aggregate Root represents a significant domain concept and is the root
 * of a cluster of domain objects that can be treated as a single unit.
 *
 * In Event Sourcing, aggregate state is reconstructed by replaying all
 * historical events.
 */
abstract class AggregateRoot
{
    /**
     * @var DomainEvent[] Uncommitted events
     */
    private array $uncommittedEvents = [];

    protected string $id;
    protected int $version = 0;

    final public function getId(): string
    {
        return $this->id;
    }

    final public function getVersion(): int
    {
        return $this->version;
    }

    /**
     * Records a new event to be committed
     *
     * @param DomainEvent $event The event to record
     */
    final protected function recordEvent(DomainEvent $event): void
    {
        $this->applyChange($event, true);
    }

    /**
     * Applies an event to aggregate state
     *
     * @param DomainEvent $event The event to apply
     * @param bool $isNew Whether to add to uncommitted events
     */
    private function applyChange(DomainEvent $event, bool $isNew): void
    {
        $this->apply($event);

        if ($isNew) {
            $this->uncommittedEvents[] = $event;
        }
    }

    /**
     * Template method for applying event to state
     *
     * Derived classes must implement this using pattern matching or reflection
     * to invoke the specific apply method for each event type.
     *
     * Example implementation:
     * ```php
     * protected function apply(DomainEvent $event): void
     * {
     *     match ($event::class) {
     *         CartCreatedEvent::class => $this->applyCartCreated($event),
     *         ProductAddedEvent::class => $this->applyProductAdded($event),
     *         default => throw new UnsupportedEventException($event::class)
     *     };
     * }
     * ```
     *
     * @param DomainEvent $event The event to apply
     */
    abstract protected function apply(DomainEvent $event): void;

    /**
     * Returns all uncommitted events
     *
     * @return DomainEvent[]
     */
    final public function getUncommittedEvents(): array
    {
        return $this->uncommittedEvents;
    }

    /**
     * Marks all events as committed
     *
     * Called after events are successfully persisted to the event store.
     */
    final public function markEventsAsCommitted(): void
    {
        $this->uncommittedEvents = [];
    }

    /**
     * Reconstructs aggregate state from event history
     *
     * This is the heart of Event Sourcing: instead of loading state from a
     * database, we replay all historical events.
     *
     * @param DomainEvent[] $history The event history
     */
    final public function loadFromHistory(array $history): void
    {
        foreach ($history as $event) {
            if (!$event instanceof DomainEvent) {
                throw new \InvalidArgumentException(
                    'All items in history must implement DomainEvent'
                );
            }

            $this->applyChange($event, false);
            $this->version++;
        }
    }
}
```

### 2. Domain Event Interface

```php
<?php

declare(strict_types=1);

namespace Muflone\Core\Events;

interface DomainEvent
{
    /**
     * Returns the aggregate ID this event belongs to
     */
    public function getAggregateId(): string;

    /**
     * Returns the event type/name
     */
    public function getEventType(): string;

    /**
     * Returns when the event occurred
     */
    public function getOccurredOn(): \DateTimeImmutable;

    /**
     * Returns the event schema version
     */
    public function getEventVersion(): int;
}
```

### 3. Command Pattern

```php
<?php

declare(strict_types=1);

namespace Muflone\Core\Commands;

interface Command
{
    /**
     * Returns the unique command ID
     */
    public function getCommandId(): string;

    /**
     * Returns when the command was created
     */
    public function getCreatedAt(): \DateTimeImmutable;
}

interface CommandHandler
{
    /**
     * Handles command execution
     *
     * @throws CommandHandlerException
     */
    public function handle(Command $command): void;
}

interface CommandBus
{
    /**
     * Dispatches a command to its handler
     *
     * @throws CommandHandlerException
     */
    public function dispatch(Command $command): void;

    /**
     * Registers a handler for a command type
     */
    public function register(string $commandClass, CommandHandler $handler): void;
}
```

### 4. Event Store Interface

```php
<?php

declare(strict_types=1);

namespace Muflone\Core\EventStore;

use Muflone\Core\Events\DomainEvent;
use Muflone\Core\Exceptions\ConcurrencyException;

interface EventStore
{
    /**
     * Saves events for an aggregate
     *
     * @param string $aggregateId
     * @param DomainEvent[] $events
     * @param int $expectedVersion For optimistic locking
     * @throws ConcurrencyException
     */
    public function saveEvents(
        string $aggregateId,
        array $events,
        int $expectedVersion
    ): void;

    /**
     * Retrieves all events for an aggregate
     *
     * @return DomainEvent[]
     */
    public function getEvents(string $aggregateId): array;

    /**
     * Retrieves events from a specific version
     *
     * @return DomainEvent[]
     */
    public function getEventsFromVersion(
        string $aggregateId,
        int $fromVersion
    ): array;
}
```

### 5. Repository Pattern

```php
<?php

declare(strict_types=1);

namespace Muflone\Core\Persistence;

use Muflone\Core\Domain\AggregateRoot;

/**
 * @template T of AggregateRoot
 */
interface Repository
{
    /**
     * Saves an aggregate
     *
     * @param T $aggregate
     * @throws RepositoryException
     */
    public function save(AggregateRoot $aggregate): void;

    /**
     * Retrieves an aggregate by ID
     *
     * @return T|null
     * @throws RepositoryException
     */
    public function getById(string $id): ?AggregateRoot;
}
```

---

## Development Workflow

### Phase 1: Analysis (1-2 hours)

**Objectives:**
1. Clone and explore the C# repository
2. Document namespace organization
3. Identify all public interfaces
4. Map dependencies between components
5. Create component responsibility matrix

**Deliverable**: `docs/original-library-analysis.md`

### Phase 2: Core Building Blocks (2-4 hours)

**Components:**
- Interfaces: `Command`, `DomainEvent`, `CommandHandler`, `EventHandler`, `Repository`
- Base classes: `AggregateRoot`, `BaseDomainEvent`, `BaseCommand`
- Value objects and entities

**For each component:**
- Complete PHPDoc documentation
- Usage examples in comments
- Unit tests with > 90% coverage

### Phase 3: Persistence Layer (3-5 hours)

**Components:**
- `EventStore` interface and in-memory implementation
- `EventSourcedRepository`
- `EventSerializer` (JSON)
- Optimistic concurrency control

**Deliverables:**
- Docker setup (PHP 8.4, Composer)
- Integration tests

### Phase 4: Messaging System (2-4 hours)

**Components:**
- `CommandBus` implementation
- `EventBus` implementation
- Handler registration system
- PSR-11 compliant DI container

**Requirements:**
- Robust error handling
- Operation logging
- Comprehensive tests

### Phase 5: Examples and Documentation (2-3 hours)

**Deliverables:**
- Complete aggregate example (e.g., ShoppingCart)
- Comprehensive README.md
- Architecture documentation
- Working examples directory
- CONTRIBUTING.md

### Phase 6: Testing and Quality Assurance (2-3 hours)

**Tasks:**
- Complete test suite
- Achieve > 80% code coverage
- PHPStan level 8 compliance
- Integration tests
- Performance benchmarks (optional)

---

## Quality Standards

### Code Quality Requirements

**MUST:**
- Strict types enabled: `declare(strict_types=1);`
- Type hints on all parameters and return values
- PHPDoc on all public methods
- Immutable value objects and events
- SOLID principles adherence

**SHOULD:**
- Cyclomatic complexity ≤ 10 per method
- Method length ≤ 50 lines
- Class cohesion (single responsibility)

### Testing Requirements

**Code Coverage:**
- Minimum: 80%
- Target: 90%+
- Critical paths: 100%

**Test Types:**
- Unit tests for all public APIs
- Integration tests for component interactions
- Functional tests for end-to-end flows
- Edge case coverage
- Error condition testing

### Static Analysis

**PHPStan Configuration:**
```neon
parameters:
    level: 8
    paths:
        - src
        - tests
    checkMissingIterableValueType: true
    checkGenericClassInNonGenericObjectType: true
    reportUnmatchedIgnoredErrors: true
```

---

## Infrastructure Setup

### Composer Configuration

```json
{
    "name": "muflone/muflone-php",
    "description": "CQRS and Event Sourcing library for PHP 8.4",
    "type": "library",
    "license": "MIT",
    "keywords": ["cqrs", "event-sourcing", "ddd", "domain-driven-design"],
    "require": {
        "php": "^8.4",
        "psr/container": "^2.0",
        "psr/log": "^3.0"
    },
    "require-dev": {
        "phpunit/phpunit": "^11.0",
        "phpstan/phpstan": "^1.10",
        "phpstan/phpstan-phpunit": "^1.3",
        "squizlabs/php_codesniffer": "^3.7"
    },
    "autoload": {
        "psr-4": {
            "Muflone\\": "src/"
        }
    },
    "autoload-dev": {
        "psr-4": {
            "Muflone\\Tests\\": "tests/"
        }
    },
    "scripts": {
        "test": "phpunit",
        "test-coverage": "phpunit --coverage-html coverage",
        "phpstan": "phpstan analyse",
        "cs-check": "phpcs",
        "cs-fix": "phpcbf"
    }
}
```

### Docker Setup

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  php:
    build:
      context: .
      dockerfile: Dockerfile
    volumes:
      - ./:/var/www/html
    working_dir: /var/www/html
    command: vendor/bin/phpunit

  php-cli:
    build:
      context: .
      dockerfile: Dockerfile
    volumes:
      - ./:/var/www/html
    working_dir: /var/www/html
    stdin_open: true
    tty: true

  phpstan:
    build:
      context: .
      dockerfile: Dockerfile
    volumes:
      - ./:/var/www/html
    working_dir: /var/www/html
    command: vendor/bin/phpstan analyse src tests --level=8
```

**Dockerfile:**
```dockerfile
FROM php:8.4-cli

RUN apt-get update && apt-get install -y \
    git \
    unzip \
    && rm -rf /var/lib/apt/lists/*

# Install Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Install PHP extensions if needed
RUN docker-php-ext-install opcache

WORKDIR /var/www/html

CMD ["php", "-a"]
```

---

## .NET to PHP Library Equivalences

### System.Collections.Generic
- **C#**: `List<T>`, `Dictionary<TKey, TValue>`
- **PHP**: Native arrays, `SplObjectStorage` if needed

### System.Threading.Tasks
- **C#**: `Task`, `async/await`
- **PHP**: Synchronous execution (document clearly)

### Newtonsoft.Json / System.Text.Json
- **C#**: JSON serialization with type metadata
- **PHP**: `json_encode()`, `json_decode()` with custom serialization helpers

### Microsoft.Extensions.DependencyInjection
- **C#**: Built-in DI container
- **PHP**: Custom PSR-11 compliant container

### System.Reflection
- **C#**: Reflection API
- **PHP**: `ReflectionClass`, `ReflectionMethod`

---

## Guiding Principles

### 1. Clarity Over Performance
- Readable and maintainable code
- Comments where necessary
- Explicit naming

### 2. Type Safety First
- Use type hints everywhere possible
- Leverage PHP 8.4 features
- PHPStan level 8 compliance

### 3. Immutability Where Possible
- Immutable value objects
- Immutable events
- Reduces side effects

### 4. SOLID Principles
- Single Responsibility
- Open/Closed
- Liskov Substitution
- Interface Segregation
- Dependency Inversion

### 5. Test-Driven Development
- Tests before implementation when possible
- Red-Green-Refactor cycle
- High coverage

---

## Final Deliverables Checklist

### Code
- [ ] Complete source code in `src/`
- [ ] Comprehensive tests in `tests/` with > 80% coverage
- [ ] All tests passing
- [ ] PHPStan level 8 passing
- [ ] PHPCS compliant

### Documentation
- [ ] Detailed README.md with examples
- [ ] Architecture documentation in `docs/`
- [ ] API documentation (PHPDoc)
- [ ] CHANGELOG.md
- [ ] CONTRIBUTING.md
- [ ] Working examples in `examples/`

### Infrastructure
- [ ] composer.json configured
- [ ] Docker setup working
- [ ] PHPStan configuration (level 8)
- [ ] PHPUnit configuration
- [ ] .gitignore appropriate

### Quality Gates
- [ ] All C# files translated
- [ ] No dependencies on Symfony/Laravel
- [ ] Follows PSR-1, PSR-4, PSR-11, PSR-12
- [ ] Every public method has complete PHPDoc

---

## Success Metrics

### Quantitative Metrics
- **Test Coverage**: > 80% (target 90%+)
- **PHPStan**: Level 8, zero errors
- **Code Quality**: Cyclomatic complexity ≤ 10
- **Documentation**: 100% public API documented
- **Performance**: Benchmark against reasonable thresholds

### Qualitative Metrics
- **Code Readability**: Clear, self-documenting code
- **Maintainability**: Easy to extend and modify
- **Usability**: Simple, intuitive API
- **Documentation Quality**: Clear examples and guides

---

## Risk Management

### Technical Risks

**Risk**: Async/await translation complexity
**Mitigation**: Use synchronous patterns with clear documentation. Consider ReactPHP/Swoole for advanced users.

**Risk**: Generic types translation
**Mitigation**: Use PHPDoc templates and runtime type checks where needed.

**Risk**: Performance compared to C#
**Mitigation**: Focus on correctness first, optimize later with benchmarks.

### Project Risks

**Risk**: Scope creep
**Mitigation**: Stick to phased approach, complete each phase before moving on.

**Risk**: Incomplete understanding of C# library
**Mitigation**: Thorough analysis phase, document uncertainties, ask for clarification.

---

## Resources and References

### CQRS and Event Sourcing
- [CQRS Journey by Microsoft](https://docs.microsoft.com/en-us/previous-versions/msp-n-p/jj554200(v=pandp.10))
- [Event Sourcing by Martin Fowler](https://martinfowler.com/eaaDev/EventSourcing.html)
- [Greg Young's CQRS Documents](https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf)

### PHP Best Practices
- [PHP The Right Way](https://phptherightway.com/)
- [PHP-FIG PSR Standards](https://www.php-fig.org/psr/)
- [PHPStan Documentation](https://phpstan.org/user-guide/getting-started)

### Testing
- [PHPUnit Documentation](https://phpunit.de/documentation.html)
- [Test Driven Development by Kent Beck](https://www.amazon.com/Test-Driven-Development-Kent-Beck/dp/0321146530)

---

## Appendix: Quick Reference

### Essential Commands

```bash
# Setup
docker-compose build
docker-compose run php composer install

# Development
docker-compose run php vendor/bin/phpunit
docker-compose run phpstan
docker-compose run php vendor/bin/phpcs

# Coverage
docker-compose run php vendor/bin/phpunit --coverage-html coverage

# Interactive Shell
docker-compose run php-cli bash
```

### File Organization Pattern

```
Component/
├── src/
│   └── ComponentName.php          # Implementation
├── tests/
│   ├── Unit/
│   │   └── ComponentNameTest.php  # Unit tests
│   └── Integration/
│       └── ComponentNameIntegrationTest.php
└── docs/
    └── component-name.md          # Component documentation
```

---

## Conclusion

This translation project is an opportunity to bring enterprise-grade CQRS/Event Sourcing patterns to the PHP ecosystem. By following this guide, maintaining high quality standards, and adhering to PHP best practices, we will create a library that serves the PHP community well while honoring the excellent work done in the original C# implementation.

**Remember**: Quality over speed. A well-tested, well-documented library that works correctly is worth more than a fast but buggy implementation.

Happy coding! 🚀
