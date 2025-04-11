---
title: "Architecting for Testability in Go (Part 3): Real-World Applications and Advanced Patterns"
date: 2025-04-10
lastmod: 2025-04-10
description: "Real-world applications, metrics, and advanced patterns for testable Go architectures"
tags: ["go", "testing", "architecture", "clean-code"]
categories: ["Software Development"]
draft: false
code: true
series: ["Go Testing Architecture"]
series_order: 3
---

{{< alert >}}
This is Part 3 of a 3-part series on architecting Go applications for testability. This final part covers real-world applications, metrics, and advanced patterns.
{{< /alert >}}

## The Power of Pure Domain Logic

In [Part 1](/posts/architecting-for-testability-in-go/part-1/), we introduced the concept of structuring applications with a pure functional core surrounded by an imperative shell. Let's dig deeper into why this pattern is so powerful for testability:

### Our Hybrid Approach: A Testable Imperative Shell

A key innovation in our approach is how we handle the "imperative shell" portion of the Functional Core, Imperative Shell pattern. In the classic FCIS pattern, the shell contains all side effects and is typically tested primarily through integration tests.

What makes our approach different is that we've made the imperative shell itself highly testable by:

1. Splitting it into two parts:
   - **Application Layer**: Handles orchestration but uses interfaces to abstract external dependencies
   - **Infrastructure Layer**: Implements those interfaces and manages actual I/O

2. Using dependency inversion to allow fake implementations to be injected for testing

This approach gives us the best of both worlds: pure domain logic that's trivially testable without mocks, and orchestration logic that can be tested without real external dependencies. This is a significant advantage over the pure FCIS pattern, while maintaining a simpler structure than traditional Hexagonal or Onion architectures.

### Pure Functions: The Testing Superpower

Domain logic implemented as pure functions with immutable data types offers extraordinary testing benefits:

```go
// Example of pure domain logic - easy to test!
package domain

// Value object with immutability
type User struct {
    ID       string
    Name     string
    Email    string
    IsActive bool
}

// Pure function - no side effects, just input → output
func CanUserAccess(user User, resourceType string, resourceID string) bool {
    if !user.IsActive {
        return false
    }
    
    // Check specific access rules...
    return checkAccessRules(user, resourceType, resourceID)
}

// Testing is trivial:
func TestCanUserAccess(t *testing.T) {
    inactiveUser := User{ID: "1", Name: "Test", IsActive: false}
    assert.False(t, CanUserAccess(inactiveUser, "document", "123"))
    
    activeUser := User{ID: "2", Name: "Test", IsActive: true}
    // More assertions...
}
```

The key characteristics that make pure domain logic easy to test:

1. **Deterministic behavior**: Same inputs always yield same outputs
2. **No hidden inputs**: All inputs are explicit parameters
3. **No side effects**: No external state modification
4. **No dependencies**: No mocking/stubbing required
5. **Immutability**: No defensive copying needed in tests

By keeping complex business rules in the domain, we minimize the imperative code in the application layer, whose primary job becomes orchestration rather than logic.

## The Investment Argument: Is This Worth It?

A common concern about the architectural approach outlined in our series is the upfront investment required. Let's address this with concrete evidence and reasoning.

### Simplicity vs. Architecture

Before diving into metrics, let's address a common criticism: "Isn't this just adding unnecessary complexity?"

Many developers, particularly those coming from enterprise .NET backgrounds, have been burned by over-architected solutions that promised maintainability but delivered complexity. They've seen codebases where:

- Simple feature additions required touching a dozen files across multiple projects
- Dependency injection configurations grew to hundreds of lines
- Repositories ballooned to hundreds of methods
- Tests became brittle verification of implementation details rather than behavior

The approach presented here deliberately aims for a "just enough" architecture that:

1. **Minimizes boilerplate**: No complex frameworks or excessive abstractions required
2. **Follows Go idioms**: Leverages interfaces and composition naturally
3. **Creates natural boundaries**: The structure feels logical, not forced
4. **Pays for itself quickly**: Initial investment in structure is rapidly offset by reduced testing and maintenance costs

Unlike some over-engineered implementations of hexagonal architecture or clean architecture, this approach focuses on pragmatic benefits rather than architectural purity. The number of layers and interfaces is kept to the practical minimum needed to achieve clear separation of concerns.

### The Value Proposition

When considering whether this architectural approach is worth the investment, consider these potential benefits:

1. **Localized Changes**: With clear boundaries between layers, changes tend to be contained within specific components, potentially making refactoring less risky.

2. **Testing Focus**: The approach enables focused testing strategies for each layer - pure function testing for domain logic and fake-based testing for application logic.

3. **Code Organization**: The structure provides a consistent place for different types of code, which may help in understanding where new code should be placed.

4. **Separation of Concerns**: By isolating domain logic from external dependencies, this approach can make it easier to reason about the business rules independently.

5. **Team Collaboration**: Clear boundaries can provide natural lines of responsibility for different team members or sub-teams.

It's important to evaluate whether these potential benefits apply to your specific context and team. The approach may not be appropriate for every project, particularly smaller utilities or applications with minimal business logic complexity.

## Comparison with Other Testing Approaches

Let's compare our fake-based approach with alternatives:

{{< figure
    src="images/testing-approaches-comparison.svg"
    alt="Comparison of different testing approaches"
    caption="Comparison of testing approaches: Mocks vs. Stubs vs. Fakes vs. Integration Tests"
>}}

| Approach | Pros | Cons | Best For |
|---|---|---|---|
| Mocks/Spies | Quick for simple cases; Fine-grained verification; Mature tooling | Brittle (implementation-focused); Hard to maintain; Test-code coupling | Verifying specific interactions when they're part of the contract |
| Stubs | Very simple; Fast; Low maintenance | Limited to canned responses; No state tracking; Limited verification | Simple scenarios with fixed responses |
| Heavy Integration | High confidence in real behavior; Tests actual integrations | Slow; Resource-intensive; Flaky; Complex setup | Critical paths; Confirming system wiring |
| Fakes + Functional Core | State/behavior focused; Refactor-friendly; Fast; Simplifies most domain testing | Initial implementation effort; Requires architectural discipline | Most business applications; Systems needing maintainability |

### When NOT to Use This Approach

This architecture may not be the optimal choice for all situations:

1. **Very Small Projects**: For simple CLIs or utilities with minimal business logic, the architectural overhead may not provide sufficient benefits.

2. **Short-lived Projects**: If the code will have a limited lifespan and maintenance isn't a primary concern.

3. **Infrastructure-focused Code**: Some types of infrastructure-heavy code (like proxies or simple data transformations) might not benefit as much from this separation.

In these cases, a simpler approach might be more appropriate, though aspects of the pattern (like clear interfaces) could still be valuable.

## Advanced Pattern: Supporting Dry-Run Mode

One powerful benefit of having well-designed fakes is the ability to use them for features like dry-run mode. This provides additional ROI on the testing investment.

```go
// Example main.go with dry-run support
import (
    "flag"
    "context"
    // ...other imports
)

func main() {
    isDryRun := flag.Bool("dry-run", false, "Execute in dry-run mode (no side effects)")
    flag.Parse()

    // Configure dependencies based on mode
    cfg := bootstrap.Config{IsDryRun: *isDryRun}
    
    // Only connect to real DB if not in dry-run
    var db *sql.DB
    if !*isDryRun {
        db = setupRealDatabase()
    }
    
    // Use factories to select appropriate implementations
    repo := bootstrap.NewRepository(ctx, cfg, db) // Returns fake if dry-run
    logger := bootstrap.NewLogger(ctx, cfg) // Could use real logger even in dry-run
    
    service := application.NewUserService(repo, logger)
    
    // Run the application...
    if *isDryRun {
        logger.Info(ctx, "--- DRY RUN MODE ENABLED ---")
    }
    
    // After operations, can inspect fake state if desired
    if *isDryRun {
        if fakeRepo, ok := repo.(*storage.FakeRepository); ok {
            for _, op := range fakeRepo.GetOperations() {
                // Log details about what would have happened
                logger.Info(ctx, "Would have performed operation", 
                    "type", op.OperationType,
                    "timestamp", op.Timestamp)
            }
        }
    }
}
```

This feature allows users to:
- Preview the effects of commands without making actual changes
- Debug complicated workflows safely
- Validate input data before committing changes

It's essentially a free bonus from the testing investment.

## Migrating Existing Codebases

Moving an existing codebase to this architecture doesn't require a complete rewrite. Here's a pragmatic migration approach:

1. **Start with Interfaces**: Begin by defining interfaces for your repositories and other infrastructure dependencies.

2. **Extract Pure Logic**: Identify business rules that can be moved to pure functions in a domain package.

3. **Implement Fakes**: Create fake implementations alongside your real ones for testing.

4. **Incremental Refactoring**: Apply this pattern to new features first, then gradually refactor existing code.

5. **Boundary Definition**: Establish clear boundaries between layers, possibly using separate packages.

Remember that imperfect progress is better than no progress. Even partial adoption of these principles can yield benefits.

## Testing Logging with Structured Analysis

Let's look at a more sophisticated approach for structured logging verification:

```go
func TestServiceLogsCorrectStructuredData(t *testing.T) {
    // Arrange
    fakeLogger := logging_test.NewFakeLogger()
    service := NewReportingService(fakeLogger)
    
    // Act
    service.GenerateReport("monthly", "2023-04")
    
    // Assert with structured analysis
    entries := fakeLogger.GetEntries()
    
    // Find specific structured log
    reportStartEntry := findLogEntry(entries, "info", "Report generation started")
    require.NotNil(t, reportStartEntry, "Missing expected log entry")
    
    // Analyze structured fields
    fields := extractKeyValuePairs(reportStartEntry.Args)
    assert.Equal(t, "monthly", fields["report_type"])
    assert.Equal(t, "2023-04", fields["period"])
    
    // Verify sequence
    completionEntry := findLogEntry(entries, "info", "Report generation completed")
    require.NotNil(t, completionEntry, "Missing completion log")
    
    // Verify completion entry was logged after start entry
    assert.Greater(t, 
        indexOfEntry(entries, completionEntry),
        indexOfEntry(entries, reportStartEntry),
        "Completion log should come after start log")
}

// Helper functions...
```

This verifies both that logging occurred and that the specific contextual data was included.

## Conclusion: The Path to More Maintainable Go Applications

Throughout this series, we've explored an architectural approach that combines:

1. **Pure Functional Core**: Business logic as pure functions with immutable data
2. **Clean Layered Architecture**: Clear separation of concerns with dependency inversion
3. **Fake-Based Testing**: State-focused testing that enables refactoring

This approach builds upon established patterns like Hexagonal Architecture (Ports and Adapters), Clean Architecture, and aspects of Domain-Driven Design, but creates a pragmatic synthesis rather than just implementing any single established pattern.

What makes this approach worth considering:
- Isolation of domain logic in pure functions (from FCIS)
- A testable application layer with interface-based dependency inversion (from Hexagonal Architecture)
- Clear ownership of external contracts by the application layer
- The use of state-based fakes rather than interaction-based mocks
- A practical approach to testing at different levels
- A simplified layer structure that avoids overengineering

### The Go Advantage

One of Go's strengths is that it's simple by design. The language itself discourages over-engineering through:

- Lack of inheritance and class hierarchies
- Structural typing with interfaces
- Minimal syntax and language features
- Focus on readability and simplicity
- Strong standard library


{{< figure
    src="images/golang-impact.svg"
    alt="The impact of Go's key features: Simplicity, Efficiency, Concurrency, Ease of Use, and Robust Library converging to enable High-Performance Applications"
    caption="Go's design principles make it ideal for implementing this architectural approach"
>}}

Go encourages straightforward, readable code. The architecture described in this series aims to provide just enough structure to support testability and maintainability while aligning with Go's philosophy of simplicity.

By focusing on essential structural elements and making them idiomatic to Go, this approach attempts to achieve the benefits of more formal architectural patterns while maintaining the simplicity that makes Go attractive in the first place.

This combination creates what many teams find to be the ideal balance between:
- Development speed (fast tests, clear boundaries)
- Maintainability (easy refactoring, isolated changes)
- Reliability (thorough testing at all levels)

The key insight is that testing problems are usually structural problems in disguise. By addressing architecture first, testing becomes a natural extension of good design rather than a burden.

As Go continues to mature as a language for business applications, these patterns help teams scale development while maintaining high quality and developer satisfaction. Whether you're building a new service or refactoring an existing one, these approaches provide a path to more maintainable, testable, and ultimately successful Go applications.
