---
title: "Architecting for Testability in Go (Part 1): Foundations and Principles"
date: 2025-04-10
lastmod: 2025-04-10
description: "Understanding the architectural foundations that enable better testing in Go applications"
tags: ["go", "testing", "architecture", "clean-code"]
categories: ["Software Development"]
draft: false
code: true
series: ["Go Testing Architecture"]
series_order: 1
---

{{< alert >}}
This is Part 1 of a 3-part series on architecting Go applications for testability. This part covers the foundational principles and architectural approach.
{{< /alert >}}

## Introduction

How easily can you understand a piece of code seconds after looking at it? How quickly can you write a meaningful test for it? 

For many engineering teams, the answer is often "not easily enough." Complex structures lead to high cognitive load, difficult reasoning, and consequently, tests that are brittle, slow, or hard to write – making refactoring daunting and slowing down development.

When tests are painful to write or maintain, it's usually a symptom of underlying architectural issues. Well-structured applications with clear separation of concerns naturally lend themselves to straightforward, "boring" tests – and boring tests are good tests.

## Relationship to Established Architectural Patterns

The approach described in this series builds upon established architectural patterns, but creates a pragmatic hybrid that addresses specific testing challenges:

- **Functional Core, Imperative Shell (FCIS)**: This pattern, popularized by Gary Bernhardt, emphasizes pure functions for business logic and relegates side effects to the outer "shell." 
  - **How we apply it**: We strictly enforce a pure functional domain with immutable types and no side effects. However, in pure FCIS, the imperative shell can be difficult to test since it contains all side effects. Our approach makes the shell testable by dividing it into an application layer (with abstractions for external dependencies) and infrastructure layer (with implementations), enabling easier testing through fakes.

  {{< figure
    src="images/functional-core-imperative-shell.svg"
    alt="Functional Core Imperative Shell"
    caption="Functional Core, Imperative Shell pattern"
>}}

- **Hexagonal/Onion Architecture**: Both patterns focus on dependency inversion where domain logic is independent of external concerns, with dependencies pointing inward through abstractions.
  - **How we apply it**: From experience, implementations of these patterns often become over-engineered with excessive abstraction layers. Our approach applies the core principles (separation of concerns, dependency inversion) while deliberately minimizing the number of layers and abstractions to what's essential for testability.

   {{< figure
    src="images/hexagonal-architecture.svg"
    alt="Hexagonal/Onion Architecture"
    caption="Hexagonal/Onion Architecture pattern"
>}}


- **Domain-Driven Design (DDD)**: While not requiring full DDD adoption, our approach embraces the separation of domain logic and the use of a ubiquitous language within bounded contexts.
  - **How we apply it**: We focus on the tactical patterns that improve testability, particularly immutable value objects and entities in our domain layer.

The architecture described here isn't claiming to be novel—rather, it's a practical synthesis of these proven patterns, creating a testing-friendly approach that maintains the benefits of each while avoiding common pitfalls like over-abstraction and excessive complexity.

### How This Approach Differs From Traditional Onion Architecture

While the architecture we'll describe in this article shares principles with Hexagonal/Onion architecture, there are important differences you'll see as we explore it in detail:

1. **Deliberate Minimalism**: Unlike many onion architecture implementations that introduce numerous layers and abstractions, our approach deliberately uses only three primary layers with minimal interfaces.

2. **Functional Core Integration**: We combine the dependency inversion of onion architecture with the pure functional approach of Functional Core, Imperative Shell - creating a stronger emphasis on immutability and side-effect isolation than typical onion architectures.

3. **Go-Idiomatic Implementation**: Rather than forcing object-oriented patterns into Go, we leverage Go's structural typing and composition to achieve clean architecture principles without the ceremony.

4. **Practical Over Pure**: We prioritize practical testability and development speed over architectural purity, avoiding the "explosion of interfaces" that often plagues onion architecture implementations.

5. **Testing Strategy Integration**: Our fake-based testing approach is integral to the architecture, not just a testing technique applied afterward.

This approach isn't claiming to reinvent architecture - it's a deliberate simplification and adaptation of established patterns with a focus on what actually matters: testable, maintainable code that's quick to develop.

As we explore this architecture in more depth in the following sections, these differences will become more apparent.

### Simplicity as a Goal

A common criticism of patterns like Hexagonal/Onion Architecture—especially in the .NET ecosystem—is that they introduce unnecessary complexity and over-engineering. Many developers have experienced codebases where these patterns were applied dogmatically, resulting in:

1. **Explosion of Interfaces**: One interface per class, often with only a single implementation
2. **Deep Inheritance Hierarchies**: Abstractions built on abstractions
3. **Bloated Projects**: Simple features requiring changes across numerous files and projects
4. **Development Friction**: Taking disproportionate time to implement straightforward functionality
5. **Testing Complexity**: Tests that focus more on verifying implementation details than behavior

This series aims to counter these experiences by presenting:

1. **A Minimalist Implementation**: We focus on the essential structural elements that deliver the most value, avoiding ceremonial abstractions.

2. **Pragmatic Boundaries**: Clear but not overly rigid separation of concerns, with just enough structure to enable testability and maintainability.

3. **Go-Idiomatic Approach**: Leveraging Go's simplicity and structural typing rather than forcing patterns from other languages.

4. **Embracing Go's Philosophy**: Go was designed to be simple and approachable. The language itself discourages over-engineering through its lack of inheritance, straightforward type system, and emphasis on clarity over cleverness. Our approach aligns with this philosophy by favoring direct, readable code over complex abstractions.

The goal is to achieve a "Goldilocks architecture" – not too complex, not too simplistic – that provides the benefits of clean separation without drowning in abstraction. The ultimate measure of success is whether the architecture makes your codebase easier to understand, test, and change over time.

## The Problem with Traditional Testing Approaches

Before diving into solutions, let's understand some challenges that can arise with common testing approaches:

### Mocks: Implementation Coupling

Testing with mocking frameworks often focuses on verifying interactions ("was method X called with argument Y?"). This approach can have several drawbacks:

* **Potential Test Brittleness**: Tests may break during refactoring, even when behavior is unchanged, if they're tied to implementation details.
* **Implementation Coupling**: Tests might resist changes to internal implementation details, potentially hindering refactoring.
* **Setup Complexity**: Mock setup and expectation management can become verbose and might obscure the test's intent.

```go
// Example of a mock-based test
func TestServiceWithMocks(t *testing.T) {
    mockRepo := new(MockRepository)
    mockRepo.On("Get", "user1").Return(user, nil)
    mockRepo.On("Save", mock.MatchedBy(func(u User) bool {
        return u.ID == "user1" && u.IsActive == true
    })).Return(nil)
    
    service := NewUserService(mockRepo)
    err := service.ActivateUser("user1")
    
    require.NoError(t, err)
    mockRepo.AssertExpectations(t)  // This verifies the implementation details
}
```

### Integration Tests: Trade-offs

Relying primarily on integration tests with real dependencies brings different considerations:

* **Execution Time**: Tests involving databases, file systems, or network calls typically take longer to run.
* **Resource Requirements**: May require additional infrastructure setup for CI/CD environments.
* **Environmental Factors**: Tests might be affected by external conditions.

## The Clean Architecture Structure: Functional Core, Imperative Shell

A clean, layered architecture separates your application into distinct areas of responsibility, closely following the "Functional Core, Imperative Shell" pattern:

```
internal/
├── domain/       # Pure business logic and types (Functional Core)
├── application/  # Orchestration logic & interface definitions
│   ├── contracts/    # DTOs defining application boundaries
│   │   ├── api/      # e.g., ActivateUserRequest, UserResponse 
│   │   └── messaging/ # e.g., UserActivatedEvent
│   └── interfaces.go # Repository, Logger, etc. interfaces
└── infrastructure/ # Interactions with outside world (Imperative Shell)
    ├── bootstrap/    # Wires dependencies together
    ├── http/         # HTTP handlers (use application/contracts/api)
    ├── messaging/    # Message handlers (use application/contracts/messaging)
    └── storage/      # Real & fake repository implementations
```

With this structure:

1. **Domain Layer (Functional Core)**: 
   - Contains pure business logic with no dependencies
   - Uses immutable data types and value objects
   - Consists of pure functions with no side effects
   - All operations are deterministic and return new values rather than modifying state
   - Easy to test in complete isolation
   - **Important**: Domain models are only used by domain and application layers - they never leak to external interfaces (APIs, messaging) or cross service boundaries

2. **Application Layer (Begins the Imperative Shell)**: 
   - Orchestrates use cases by combining domain logic with external effects
   - Defines interfaces for what it needs from the outside world
   - Manages workflow and translates between domain and external concerns
   - **Owns all contract definitions** (API DTOs, message schemas) through which the outside world interacts with the system
   - Performs mapping between domain models and external contracts to prevent domain leakage

3. **Infrastructure Layer (Completes the Imperative Shell)**: 
   - Handles all side effects (database, network, etc.)
   - Provides entry points into the application via APIs, message consumers, CLI commands, etc.
   - Implements interfaces defined by the application layer
   - Contains adapters for external systems and frameworks
   - Translates between external formats (HTTP, message queues) and application contracts
   - **Never directly exposes domain models** to the outside world

{{< figure
    src="images/clean-architecture-layers.svg"
    alt="Clean Architecture Layers visualization"
    caption="Three-layer architecture with Domain (pure functions), Application (orchestration), and Infrastructure (side effects) - following the Functional Core, Imperative Shell pattern"
>}}

## Why Architect This Way?

This architectural approach offers several key benefits:

* **Reasoning About Code**: Clear boundaries make it easier to understand what each part of the system does.
* **Pure Domain Logic**: By keeping domain logic pure (no side effects, immutable data types), we gain deterministic behavior that's trivial to test in isolation.
* **Testability at All Levels**: 
  * Domain logic tests become simple input/output verifications
  * Application logic can be tested with fakes for external dependencies
  * Infrastructure components can be tested independently
* **Flexibility**: Implementation details can change without affecting the core.
* **Maintainability**: Changes tend to be localized to specific layers.
* **Explicit Side Effects**: By pushing all side effects to the outer layers, we make them explicit and easier to manage.

## What Are Fake Implementations?

A "Fake" in this context is a lightweight, in-memory implementation of an interface used specifically for testing:

* A working implementation that maintains its own state (e.g., data in a map)
* Records operations for verification purposes
* Can be configured to simulate specific scenarios, including errors

Unlike traditional mocks which focus on interaction verification, fakes allow tests to verify the outcome of operations (state), making them more resilient to refactoring.

```go
// Simple example of a fake repository
type FakeRepository struct {
    entities map[string]Entity
    operations []string
}

func (f *FakeRepository) Get(id string) (Entity, error) {
    f.operations = append(f.operations, "Get:"+id)
    entity, exists := f.entities[id]
    if !exists {
        return Entity{}, errors.New("not found")
    }
    return entity, nil
}

func (f *FakeRepository) Save(entity Entity) error {
    f.operations = append(f.operations, "Save:"+entity.ID)
    f.entities[entity.ID] = entity
    return nil
}
```

## Coming Up in Part 2

In the next part of this series, we'll dive into practical implementation details with comprehensive code examples:

* Defining clear interfaces in the application layer
* Creating real and fake implementations
* Testing strategies using fakes
* Practical code examples

Continue reading [Part 2: Implementation Strategy and Code Examples](/posts/architecting-for-testability-in-go/part-2/)