# Muflone Project Overview

This document provides a comprehensive overview of the Muflone project, its architecture, and development conventions.

## Project Purpose

Muflone is a .NET library that provides a framework for implementing applications using the Command Query Responsibility Segregation (CQRS) and Event Sourcing (ES) patterns. It is based on the principles of Domain-Driven Design (DDD).

The core components of the framework include:

*   **Aggregates:** The `AggregateRoot` class is the base for domain objects that enforce business rules and invariants.
*   **Commands:** Represent the intent to change the state of an aggregate.
*   **Events:** Represent the changes in the state of an aggregate.
*   **Command Handlers:** Process commands and interact with aggregates.
*   **Event Handlers:** React to events and can be used to update read models or trigger other processes.
*   **Event Bus:** A mechanism for publishing and subscribing to events.
*   **Repository:** For persisting and retrieving aggregates.

## Building and Running

### Build

To build the project, run the following command from the `src` directory:

```sh
dotnet build
```

### Test

To run the tests, run the following command from the `src` directory:

```sh
dotnet test
```

## Development Conventions

### Coding Style

*   The project follows the standard C# coding conventions.
*   Asynchronous methods are used extensively and are named with the `Async` suffix.
*   The project uses a consistent naming convention for classes and interfaces (e.g., `ICommandHandlerAsync`, `CommandHandlerAsync`).

### Testing

*   The project has a comprehensive suite of unit and integration tests.
*   Tests are written using the xUnit framework.
*   The tests are located in the `Muflone.Tests` project.
*   The tests provide excellent examples of how to use the framework.

### Project Structure

The project is organized into the following main directories:

*   `src/Muflone`: The main project containing the core framework code.
*   `src/Muflone.Tests`: The project containing the tests for the framework.

The `Muflone` project is further organized by feature:

*   `Core`: Contains the core building blocks of the framework, such as `AggregateRoot`, `DomainId`, and `Entity`.
*   `Messages`: Contains the base classes for `Commands` and `Events`.
*   `Persistence`: Contains the interfaces for the `IRepository` and `IServiceBus`.
*   `Factories`: Contains interfaces for creating handlers.
