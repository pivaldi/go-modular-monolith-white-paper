# Testing Strategy

Test at multiple levels to ensure correctness and maintainability.

## Testing Pyramid

```
      /\      E2E Tests (few, slow)
     /  \
    /____\    Contract Tests (some, medium)
   /      \
  /________\  Integration Tests (more, faster)
 /          \
/____________\ Unit Tests (most, fastest)
```

## 1. Unit Tests (Domain Layer)

**Focus:** Business logic in isolation

```go
// services/servicebsvc/internal/domain/user/user_test.go
func TestUser_ChangePassword(t *testing.T) {
    u := user.NewUser("user-123",
        user.MustNewEmail("user@example.com"),
        user.MustHashPassword("oldpass123"),
    )

    err := u.ChangePassword("oldpass123", "newpass456")

    require.NoError(t, err)
    assert.True(t, u.VerifyPassword("newpass456"))
    assert.False(t, u.VerifyPassword("oldpass123"))
}
```

**Characteristics:**
- Very fast (<1ms per test)
- No external dependencies
- No mocks needed
- High coverage on business rules

## 2. Use Case Tests (Application Layer)

**Focus:** Orchestration logic with mocked ports

```go
// services/servicebsvc/internal/application/command/login_test.go
func TestLoginCommand_Execute(t *testing.T) {
    // Setup mocks
    userRepo := mocks.NewUserRepository(t)
    sessionRepo := mocks.NewSessionRepository(t)
    serviceaClient := mocks.NewServiceAClient(t)

    // Mock expectations
    userRepo.On("FindByEmail", ctx, testEmail).Return(testUser, nil)
    sessionRepo.On("Save", ctx, mock.Anything).Return(nil)
    serviceaClient.On("GetServiceA", ctx, "user-123").Return(&ports.ServiceAInfo{
        Name: "John Doe",
    }, nil)

    // Execute
    cmd := command.NewLoginCommand(userRepo, sessionRepo, serviceaClient, logger)
    result, err := cmd.Execute(ctx, command.LoginInput{
        Email:    "user@example.com",
        Password: "password123",
    })

    // Assert
    require.NoError(t, err)
    assert.NotEmpty(t, result.Token)
    assert.Equal(t, "John Doe", result.ServiceAName)

    userRepo.AssertExpectations(t)
}
```

**Characteristics:**
- Fast (~1-10ms per test)
- Tests use case logic
- Mocked external dependencies
- Tests error handling and edge cases

## ⚠️ The Mockist Trap: What NOT to Mock

**Common pitfall:** Generating mocks for every interface and using them in all tests leads to **fragile, implementation-coupled tests** that break on every refactoring.

### The Golden Rule: Mock Architectural Boundaries Only

**✅ DO Mock (Ports - Infrastructure Boundaries):**
- **Database access**: `TodoRepository`, `UserRepository`, `UnitOfWork`
- **External APIs**: `PaymentGateway`, `EmailService`, `GithubClient`
- **Message brokers**: `EventDispatcher`, `KafkaProducer`
- **Non-deterministic systems**: `Clock`, `RandomGenerator`, `IDGenerator`

These cross architectural boundaries into infrastructure. They're slow, have external state, or introduce non-determinism.

**❌ DON'T Mock (Internal Business Logic):**
- **Domain entities/value objects**: `Todo`, `User`, `Email`, `Priority` - Just instantiate them!
- **Application services**: `TodoService`, `UserService` - Use real implementations with mocked ports
- **In-memory operations**: Pure calculation services, formatters, validators
- **Standard library types**: `io.Reader`, `io.Writer` - Use `bytes.Buffer` or `strings.Reader`

### Why Over-Mocking Breaks Tests

When you mock internal services, you test **implementation details** instead of **behavior**:

```go
// ❌ BAD: Testing implementation details
func TestHandler_CreateTodo_OverMocked(t *testing.T) {
    mockService := mocks.NewMockTodoService(t)

    // This test breaks when you refactor the service internals!
    mockService.EXPECT().
        CreateTodo(ctx, mock.Anything).
        Return(&response, nil)

    handler := NewHandler(mockService)
    // Now you're only testing that handler calls service - no real logic tested!
}

// ✅ GOOD: Testing actual behavior through real service
func TestHandler_CreateTodo_Behavioral(t *testing.T) {
    // Mock only the PORTS (architectural boundaries)
    mockRepo := mocks.NewMockTodoRepository(t)
    mockUoW := mocks.NewMockUnitOfWork(t)
    mockDispatcher := mocks.NewMockEventDispatcher(t)

    mockUoW.EXPECT().WithTransaction(...).RunAndReturn(...)
    mockRepo.EXPECT().Save(...).Return(nil)
    mockDispatcher.EXPECT().Dispatch(...).Return(nil)

    // Use REAL service - test the actual business logic!
    service := application.NewTodoService(mockRepo, mockUoW, mockDispatcher)
    handler := NewHandler(service)

    // Tests handler → service → ports flow with real logic
}
```

### The "Fake" Alternative

For complex scenarios, write simple **in-memory fakes** instead of generated mocks:

```go
// internal/adapters/repository/memory/todo_repository.go
type InMemoryTodoRepository struct {
    todos map[string]*domain.Todo
    mu    sync.RWMutex
}

func (r *InMemoryTodoRepository) Save(ctx context.Context, todo *domain.Todo) error {
    r.mu.Lock()
    defer r.mu.Unlock()
    r.todos[todo.ID()] = todo
    return nil
}

// Use in tests - behaves like real database, no .EXPECT() needed!
```

**Benefits of Fakes:**
- Behave like real implementations
- No brittle `.EXPECT()` setup in every test
- Can be reused across test suites
- Useful for local development without Docker

### Summary: Mock Configuration

Your `mockery.yaml` should **ONLY** include infrastructure ports:

```yaml
# ✅ CORRECT - Only mock architectural boundaries
packages:
  github.com/yourorg/app/internal/application/ports:
    interfaces:
      TodoRepository: {}      # Database boundary
      UnitOfWork: {}         # Transaction boundary
      EventDispatcher: {}    # Message broker boundary

# ❌ WRONG - Don't add these:
#  github.com/yourorg/app/internal/application:
#    interfaces:
#      TodoService: {}       # This is internal logic!
```

**Key insight:** Mocking is for *architectural boundaries*, not *code organization*. Mock what's slow, external, or non-deterministic. Use real implementations for everything else.

## 3. Integration Tests (Adapters)

**Focus:** Adapters with real external systems

```go
// services/servicebsvc/internal/adapters/outbound/persistence/postgres/user_repository_test.go
func TestUserRepository_Save(t *testing.T) {
    if testing.Short() {
        t.Skip("Skipping integration test")
    }

    db := integration.SetupTestDB(t)
    defer integration.TeardownTestDB(t, db)

    repo := postgres.NewUserRepository(db)
    u := user.NewUser("user-123",
        user.MustNewEmail("user@example.com"),
        user.MustHashPassword("password123"),
    )

    err := repo.Save(context.Background(), u)
    require.NoError(t, err)

    found, err := repo.FindByID(context.Background(), "user-123")
    require.NoError(t, err)
    assert.Equal(t, u.Email(), found.Email())
}
```

**Characteristics:**
- Medium speed (~10-100ms per test)
- Real database/cache/external systems
- Tests adapter implementation
- Can use test containers

## 4. Contract Tests (Contract Definition/Service Boundaries)

**Focus:** Verify that a service correctly implements its own public API contract

**Important:** Contract tests live in the **PROVIDER** service, not the consumer. Each service tests its own implementation of its contract interface.

```go
// services/serviceasvc/test/contract/contracts_test.go
func TestServiceAService_ContractImplementation(t *testing.T) {
    // Setup: Initialize serviceasvc with test data
    // This is serviceasvc testing its OWN implementation
    app := setupTestServiceAService(t)
    defer app.Cleanup()

    // Create InprocServer (the contract implementation serviceasvc provides)
    server := inbound_contracts.NewInprocServer(
        app.GetServiceAQuery,
        app.CreateServiceACommand,
        // ... other dependencies
    )

    // Test that it correctly implements serviceasvc.ServiceAService interface
    servicea, err := server.GetServiceA(context.Background(), "test-servicea-123")
    require.NoError(t, err)
    assert.Equal(t, "Test Service A", servicea.Name)

    // Test error contract
    _, err = server.GetServiceA(context.Background(), "nonexistent")
    assert.ErrorIs(t, err, serviceasvc.ErrServiceANotFound)

    // Test DTO mapping (domain entity → contract DTO)
    assert.NotEmpty(t, servicea.ID)
    assert.NotEmpty(t, servicea.CreatedAt)
}
```

**Characteristics:**
- Tests that the service correctly implements its contract interface
- Validates DTO mapping (domain entities → contract DTOs)
- Validates error translation (domain errors → contract errors)
- Lives in the PROVIDER service, never in the consumer
- Consumer never imports provider's implementation

## 5. End-to-End Tests

**Focus:** Full user scenarios

```go
// test/e2e/user_registration_test.go
func TestUserRegistrationFlow(t *testing.T) {
    env := helpers.StartE2EEnvironment(t)
    defer env.Stop()

    // Register
    registerResp, err := env.AuthClient.Register(ctx, "user@example.com", "SecurePass123!")
    require.NoError(t, err)

    // Create Service A profile
    serviceaResp, err := env.ServiceAClient.CreateServiceA(ctx, "Jane Smith", "Writer")
    require.NoError(t, err)

    // Login
    loginResp, err := env.ServiceBClient.Login(ctx, "user@example.com", "SecurePass123!")
    require.NoError(t, err)
    assert.Equal(t, "Jane Smith", loginResp.ServiceAName)
}
```

**Characteristics:**
- Tests complete flows
- All services running
- Real interactions

## Running Tests

```bash
# Unit tests only (fast)
go test ./... -short

# All tests
go test ./...

# With coverage
go test ./... -coverprofile=coverage.out
go tool cover -html=coverage.out

# Specific service
cd services/serviceasvc
go test ./...

# Integration tests only
go test ./... -run Integration

# Via mise
mise run test:unit
mise run test:integration
mise run test:all
```
