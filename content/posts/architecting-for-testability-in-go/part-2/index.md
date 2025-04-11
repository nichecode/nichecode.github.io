---
title: "Architecting for Testability in Go (Part 2): Implementation Strategy and Code Examples"
date: 2025-04-10T02:00:00
lastmod: 2025-04-10T02:00:00
description: "Practical implementation details and code examples for testable Go applications"
tags: ["go", "testing", "architecture", "clean-code"]
categories: ["Software Development"]
draft: false
series: ["Go Testing Architecture"]
series_order: 2
---



{{< alert >}}
This is Part 2 of a 3-part series on architecting Go applications for testability. This part provides concrete implementation details and code examples.
{{< /alert >}}

## TL;DR
This part provides practical implementation details and code examples for our testable architecture approach. We'll cover interface definition, real implementations, fake implementations, and testing patterns, with complete code examples you can adapt for your projects.

## From Theory to Practice

In [Part 1](/posts/architecting-for-testability-in-go/part-1/), we established the theoretical foundations of our approach. Now it's time to see how these principles translate into actual Go code.

Our implementation strategy follows these key steps:

1. Define clear interfaces in the application layer
2. Create real implementations in the infrastructure layer
3. Create comprehensive fake implementations for testing
4. Write tests using these fakes
5. Use a factory pattern to select the appropriate implementation

Let's walk through each step with concrete code examples.

## Implementation Strategy for Fake-Based Testing

In [Part 1](/posts/architecting-for-testability-in-go/part-1/), we laid out the architectural foundations for testable Go applications. Now, let's dive into concrete implementation details with comprehensive code examples.

### 1. Define Clear Interfaces in the Application Layer

The foundation of our approach is defining interfaces in the application layer that express what the application needs, not how those needs are implemented:

```go
package application

import (
    "context"
    "myapp/internal/domain"
)

// Repository defines storage operations needed by the service.
// Why interfaces? To decouple application logic from specific storage tech.
type Repository interface {
    Get(ctx context.Context, id string) (domain.Entity, error)
    Save(ctx context.Context, entity domain.Entity) error
}

// Logger defines logging capabilities needed.
type Logger interface {
    Info(ctx context.Context, msg string, args ...any)
    Error(ctx context.Context, msg string, args ...any)
    With(args ...any) Logger // For creating contextual loggers
}
```

The key insight: the application defines what it needs, not how those needs are met. This inversion of dependencies is crucial for testability.

While the application layer contains imperative code that orchestrates workflows, all complex business rules and logic should be delegated to the pure functional domain layer. This keeps the application layer focused on coordination rather than complex logic.

It's important to note that the infrastructure layer serves two vital roles in this architecture:
1. It implements the interfaces defined by the application layer (like Repository, Logger)
2. It provides the entry points that invoke the application layer's services (HTTP handlers, message consumers, CLI commands)

This dual role makes the infrastructure layer the bridge between the outside world and your core application logic.

## Decoupling from External Contracts

A critical aspect of this architecture is protecting the domain from external contract dependencies:

```go
// application/contracts/api/request.go
package api

// UserActivationRequest is an API contract, owned by the application layer
type UserActivationRequest struct {
    UserID string `json:"user_id"`
    Reason string `json:"reason,omitempty"`
}

// application/service.go
package application

import (
    "myapp/internal/application/contracts/api"
    "myapp/internal/domain"
)

func (s *UserService) ActivateUser(ctx context.Context, req api.UserActivationRequest) error {
    // Map from API contract to domain model
    userID := req.UserID
    
    // Get domain entity from storage
    entity, err := s.repo.Get(ctx, userID)
    if err != nil {
        return err
    }
    
    // Use domain logic to perform activation (pure function)
    activatedEntity := domain.ActivateUser(entity, req.Reason)
    
    // Save updated entity
    return s.repo.Save(ctx, activatedEntity)
}

// infrastructure/http/handler.go
package http

import (
    "myapp/internal/application"
    "myapp/internal/application/contracts/api"
)

func (h *UserHandler) ActivateUser(w http.ResponseWriter, r *http.Request) {
    // Parse request into API contract
    var req api.UserActivationRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
    
    // Call application service with API contract
    err := h.userService.ActivateUser(r.Context(), req)
    if err != nil {
        // Handle error
        return
    }
    
    // Send response
    w.WriteHeader(http.StatusOK)
}
```

This separation is critical because:

1. **Domain Purity**: Domain models remain uncontaminated by API/messaging concerns (like JSON tags, validation rules)
2. **Contract Ownership**: The application layer completely owns all external contracts
3. **Mapping Control**: The application layer handles all translation between external contracts and domain models
4. **Package Structure Enforcement**: The structure makes it impossible for external concerns to leak into the domain due to import paths
5. **Compatibility Management**: API or message format changes can be handled in the application layer without affecting domain logic
6. **Domain Model Protection**: Domain models are only exposed to the application layer and persistence - they never cross external boundaries or service boundaries directly

Whether contracts live in `internal/application/contracts/...` or a shared `pkg/` directory, the key principle remains: domain models never leak outside and external contracts never leak in.

### 2. Create Real Implementations in the Infrastructure Layer

Now let's implement these interfaces with concrete types:

```go
package storage

import (
    "context"
    "database/sql"
    "myapp/internal/application"
    "myapp/internal/domain"
)

// PostgresRepository implements application.Repository using Postgres.
type PostgresRepository struct { 
    db *sql.DB 
}

func NewPostgresRepository(db *sql.DB) *PostgresRepository { 
    return &PostgresRepository{db: db} 
}

func (r *PostgresRepository) Get(ctx context.Context, id string) (domain.Entity, error) { 
    // Implementation using SQL queries with the DB connection
    // ...
    return domain.Entity{}, nil 
}

func (r *PostgresRepository) Save(ctx context.Context, entity domain.Entity) error { 
    // Implementation using SQL queries
    // ...
    return nil 
}

// Compile-time verification that PostgresRepository implements Repository
var _ application.Repository = (*PostgresRepository)(nil)
```

Similar implementations would exist for other interfaces like `Logger` with a real logging implementation.

### 3. Create Comprehensive Fake Implementations

Here's where the magic happens - creating fake implementations that are robust enough for thorough testing:

```go
package storage

import (
    "context"
    "fmt"
    "myapp/internal/application"
    "myapp/internal/domain"
    "myapp/internal/ctxkeys" // For context key handling
    "sync"
    "time"
)

// FakeRepository implements application.Repository for testing and dry-run.
type FakeRepository struct {
    mu         sync.RWMutex
    entities   map[string]domain.Entity
    operations []OperationLog // Records all operations for verification
    saveErr    error          // For simulating errors
    getErr     error
}

// OperationLog records details about calls made to the fake.
type OperationLog struct {
    OperationType string         // "Get", "Save", etc.
    Params        OperationParams
    Timestamp     time.Time
    CorrelationID string // Example of capturing context data
}

// OperationParams uses a union-like approach for type safety.
type OperationParams struct {
    GetParams  *GetParams  // Only one of these will be non-nil
    SaveParams *SaveParams // depending on operation type
}

type GetParams struct {
    ID string
}

type SaveParams struct {
    Entity domain.Entity
}

func NewFakeRepository() *FakeRepository {
    return &FakeRepository{
        entities:   make(map[string]domain.Entity),
        operations: make([]OperationLog, 0),
    }
}

// Get implements the Repository interface
func (f *FakeRepository) Get(ctx context.Context, id string) (domain.Entity, error) {
    f.mu.Lock()
    defer f.mu.Unlock()
    
    // Extract correlation ID from context as an example
    corrID := ctxkeys.GetCorrelationID(ctx)
    
    // Record this operation
    f.operations = append(f.operations, OperationLog{
        OperationType: "Get",
        Params: OperationParams{
            GetParams: &GetParams{ID: id},
        },
        Timestamp:     time.Now(),
        CorrelationID: corrID,
    })

    // Simulate error if configured
    if f.getErr != nil { 
        return domain.Entity{}, f.getErr 
    }
    
    // Otherwise return the entity from the in-memory map
    entity, ok := f.entities[id]
    if !ok { 
        return domain.Entity{}, fmt.Errorf("entity %s not found", id) 
    }
    return entity, nil
}

// Save implements the Repository interface 
func (f *FakeRepository) Save(ctx context.Context, entity domain.Entity) error {
    f.mu.Lock()
    defer f.mu.Unlock()
    
    corrID := ctxkeys.GetCorrelationID(ctx)
    
    f.operations = append(f.operations, OperationLog{
        OperationType: "Save",
        Params: OperationParams{
            SaveParams: &SaveParams{Entity: entity},
        },
        Timestamp:     time.Now(),
        CorrelationID: corrID,
    })

    if f.saveErr != nil { 
        return f.saveErr 
    }
    
    f.entities[entity.ID] = entity
    return nil
}

// Test Helper Methods (for convenient test setup and verification)
func (f *FakeRepository) WithEntities(entities ...domain.Entity) *FakeRepository {
    f.mu.Lock()
    defer f.mu.Unlock()
    for _, e := range entities {
        f.entities[e.ID] = e
    }
    return f
}

func (f *FakeRepository) SimulateGetError(err error) *FakeRepository {
    f.mu.Lock()
    defer f.mu.Unlock()
    f.getErr = err
    return f
}

func (f *FakeRepository) SimulateSaveError(err error) *FakeRepository {
    f.mu.Lock()
    defer f.mu.Unlock()
    f.saveErr = err
    return f
}

func (f *FakeRepository) GetOperations() []OperationLog {
    f.mu.RLock()
    defer f.mu.RUnlock()
    // Return a copy to prevent modification issues
    opsCopy := make([]OperationLog, len(f.operations))
    copy(opsCopy, f.operations)
    return opsCopy
}

// Ensure the fake implements the interface
var _ application.Repository = (*FakeRepository)(nil)
```

This `FakeRepository` is comprehensive:
- It maintains state in memory (the `entities` map)
- It records all operations for verification
- It can be configured to simulate errors
- It includes helper methods for test setup
- It handles thread safety with mutex locks

Similarly, we can implement a fake logger:

```go
package logging_test

import (
    "context"
    "myapp/internal/application"
    "sync"
)

// LogEntry captures details of a log call.
type LogEntry struct {
    Level   string
    Message string
    Args    []any
}

// FakeLogger implements application.Logger for testing.
type FakeLogger struct {
    mu      sync.RWMutex
    entries []LogEntry
}

func NewFakeLogger() *FakeLogger {
    return &FakeLogger{entries: make([]LogEntry, 0)}
}

func (f *FakeLogger) Info(ctx context.Context, msg string, args ...any) {
    f.log("info", msg, args)
}

func (f *FakeLogger) Error(ctx context.Context, msg string, args ...any) {
    f.log("error", msg, args)
}

func (f *FakeLogger) With(args ...any) application.Logger {
    // For simplicity, this example just returns self
    // A more sophisticated fake might track these args
    return f
}

func (f *FakeLogger) log(level string, msg string, args []any) {
    f.mu.Lock()
    defer f.mu.Unlock()
    // Copy args to prevent modification issues
    argsCopy := make([]any, len(args))
    copy(argsCopy, args)
    f.entries = append(f.entries, LogEntry{
        Level:   level,
        Message: msg,
        Args:    argsCopy,
    })
}

// Helper methods for tests
func (f *FakeLogger) Count() int {
    f.mu.RLock()
    defer f.mu.RUnlock()
    return len(f.entries)
}

func (f *FakeLogger) GetEntries() []LogEntry {
    f.mu.RLock()
    defer f.mu.RUnlock()
    // Return a copy
    entriesCopy := make([]LogEntry, len(f.entries))
    copy(entriesCopy, f.entries)
    return entriesCopy
}

func (f *FakeLogger) ContainsMessage(substr string) bool {
    f.mu.RLock()
    defer f.mu.RUnlock()
    for _, entry := range f.entries {
        if strings.Contains(entry.Message, substr) {
            return true
        }
    }
    return false
}

var _ application.Logger = (*FakeLogger)(nil)
```

### 4. Where Should Fake Implementations Live?

The placement of fake implementations depends on their purpose:

- **For testing only**: Place in `*_test.go` files within the package of the real implementation
- **For testing AND features** (like dry-run mode): Place in regular `.go` files

This ensures that test-only code doesn't bloat your production binary.

{{< alert >}}
**Testing Philosophy**: When using fakes, our tests verify state and behavior outcomes rather than implementation details. This is fundamentally different from mock-based testing, which often focuses on verifying specific method calls were made in a specific order.
{{< /alert >}}

### 5. Writing Tests with Fakes

Now that we have our fakes, let's see how to write tests with them:

```go
package application_test

import (
    "context"
    "errors"
    "myapp/internal/application"
    "myapp/internal/domain"
    "myapp/internal/ctxkeys"
    "myapp/internal/infrastructure/storage"
    "myapp/internal/infrastructure/logging_test" // Import fake logger
    "testing"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestUserService_ActivateUser_Success(t *testing.T) {
    // Arrange: Setup fakes with desired state
    ctx := context.WithValue(context.Background(), ctxkeys.CorrelationIDKey, "test-corr-id-123")
    initialEntity := domain.Entity{ID: "user1", IsActive: false, Name: "Test User"}
    
    fakeRepo := storage.NewFakeRepository().WithEntities(initialEntity)
    fakeLogger := logging_test.NewFakeLogger()
    
    service := application.NewUserService(fakeRepo, fakeLogger)
    userIDToActivate := "user1"

    // Act: Execute the application logic
    err := service.ActivateUser(ctx, userIDToActivate)

    // Assert: Verify outcome via fake state
    require.NoError(t, err)

    // Verify state change in Repository
    updatedEntity, getErr := fakeRepo.Get(ctx, userIDToActivate)
    require.NoError(t, getErr)
    assert.True(t, updatedEntity.IsActive, "Entity should be active")
    assert.Equal(t, initialEntity.Name, updatedEntity.Name, "Other fields should be unchanged")

    // Verify operations were recorded
    ops := fakeRepo.GetOperations()
    require.Len(t, ops, 2, "Expected Get and Save operations")
    
    // Verify logging occurred
    assert.GreaterOrEqual(t, fakeLogger.Count(), 1, "Should have logged at least one message")
    assert.True(t, fakeLogger.ContainsMessage("User activated"), "Expected activation message")
}

func TestUserService_ActivateUser_RepoError(t *testing.T) {
    // Arrange
    ctx := context.Background()
    repoErr := errors.New("db connection lost")
    
    fakeRepo := storage.NewFakeRepository().
        WithEntities(domain.Entity{ID: "user1", IsActive: false}).
        SimulateSaveError(repoErr) // Configure fake to simulate error
        
    fakeLogger := logging_test.NewFakeLogger()
    service := application.NewUserService(fakeRepo, fakeLogger)

    // Act
    err := service.ActivateUser(ctx, "user1")

    // Assert
    require.Error(t, err)
    assert.ErrorIs(t, err, repoErr)
    
    // Verify error was logged
    assert.True(t, fakeLogger.ContainsMessage("Failed to save"), 
        "Error should have been logged")
}
```

The key differences from traditional mock-based tests:

1. **State-based verification**: We check the outcome (entity is now active) rather than implementation details
2. **Fluent configuration**: Fakes have builder-style methods for easy setup
3. **Resilience to refactoring**: The test doesn't break if internal implementation changes
4. **Realistic behavior**: Fakes implement the same interface as real components

## Testing Logging with Fakes

Testing logging is often overlooked, but it's a crucial part of application behavior, especially for diagnostics and audit trails. Our `FakeLogger` makes this straightforward:

```go
func TestLoggingBehavior(t *testing.T) {
    // Arrange
    fakeLogger := logging_test.NewFakeLogger()
    service := NewServiceThatLogs(fakeLogger)
    
    // Act
    service.DoSomethingImportant("test-value")
    
    // Assert
    entries := fakeLogger.GetEntries()
    
    // Find specific log messages
    found := false
    for _, entry := range entries {
        if entry.Level == "info" && entry.Message == "Operation started" {
            found = true
            // Check args contain our value
            for i := 0; i < len(entry.Args); i += 2 {
                if i+1 < len(entry.Args) && 
                   entry.Args[i] == "input" && 
                   entry.Args[i+1] == "test-value" {
                    return // Test passes
                }
            }
        }
    }
    
    t.Fatalf("Expected log message not found or missing expected arguments")
}
```

This verifies both that logging occurred and that the specific contextual data was included.

## Factory Pattern for Implementation Selection

To manage which implementation (real or fake) is used, we can use a factory pattern:

```go
package bootstrap

import (
    "context"
    "database/sql"
    "myapp/internal/application"
    "myapp/internal/infrastructure/storage"
)

type Config struct {
    IsDryRun bool
    // Other config fields
}

// NewRepository creates a repository based on configuration.
func NewRepository(ctx context.Context, cfg Config, db *sql.DB) application.Repository {
    if cfg.IsDryRun {
        return storage.NewFakeRepository()
    }
    return storage.NewPostgresRepository(db)
}

// Similar factories for Logger, Notifier, etc.
```

This enables features like dry-run mode while keeping the selection logic isolated.

## Test Classification Strategy

With this architecture, we can employ a balanced testing strategy:

{{< figure
    src="images/test-pyramid.svg"
    alt="Test Pyramid showing the proportion of different test types"
    caption=""
>}}

1. **Unit Tests (~85-90%)**
   - Domain Logic Tests: Test pure business logic directly
   - Service Tests (with Fakes): Test application service orchestration
   - Handler Tests (with Fake Services): Test HTTP/messaging handlers

2. **Integration Tests (~5-10%)**
   - Test real infrastructure components against real dependencies
   - Focused on critical interfaces between components

3. **End-to-End Tests (~1-2%)**
   - Test complete user journeys through the system
   - Fewer, focused on key scenarios



## Directory Structure for Tests

Organizing tests logically helps maintainability:

```
internal/
├── application/
│   ├── service.go
│   ├── service_test.go       # Service Tests (using fakes)
│   └── interfaces.go
├── domain/
│   ├── model.go
│   └── model_test.go         # Domain Unit Tests
└── infrastructure/
    ├── http/
    │   ├── handler.go
    │   └── handler_test.go   # Handler Unit Tests
    ├── logging/
    │   ├── logger.go
    │   └── fake_logger_test.go # Fake Logger for tests
    └── storage/
        ├── postgres_repository.go
        ├── fake_repository.go # OR fake_repository_test.go
        └── postgres_repository_integration_test.go

test/
└── integration/              # Broader integration tests
    ├── setup/                # Test setup utilities
    └── api_test.go           # Full slice tests
```

Recommended placement:
- **Domain tests:** In the domain package
- **Service tests:** In application_test package
-**Handler tests:** In infrastructure/http_test package
- **Fake implementations:** In *_test.go files or regular .go files if used for features
- **Integration tests:** In a separate test/ directory or with build tags

## Coming Up in Part 3

In the final part of this series, we'll explore:
- The value proposition of this architectural approach
- Comparison with other testing methodologies
- Advanced patterns like dry-run mode
- Migration strategies for existing codebases
- When this approach might not be appropriate

Continue reading [Part 3: Real-World Applications and Advanced Patterns](/posts/architecting-for-testability-in-go/part-3/)