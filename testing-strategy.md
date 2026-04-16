# Testing Strategy

Test at multiple levels to ensure correctness and maintainability.

## Testing Pyramid

```

         /\      System Tests — ONE suite at the monolith level
        /  \     (real binary, testcontainers postgres, full auth flow,
       /    \     cross-module scenarios)
      /______\
     /        \   Contract Tests — per module, fast, in-memory
    /__________\
   /            \  Integration Tests — per module, testcontainers, adapter only
  /______________\
 /                \ Unit + Application Tests — per module, pure in-memory
/__________________\
```

## 1. Unit Tests (Domain Layer)

**Focus:** Business logic in isolation

```go
// modules/todo/internal/domain/todo_test.go
func TestTodo_Complete(t *testing.T) {
    title, _ := domain.NewTitle("Buy milk")
    todo, _ := domain.NewTodo(title, domain.EmptyDescription, nil, uuid.New())

    err := todo.Complete()

    require.NoError(t, err)
    assert.Equal(t, domain.StatusCompleted, todo.Status())
    assert.Len(t, todo.Events(), 2) // TodoCreated + TodoCompleted
}

func TestTodo_Complete_AlreadyCompleted(t *testing.T) {
    todo := mustCreateCompletedTodo(t)

    err := todo.Complete()

    assert.ErrorIs(t, err, domain.ErrCannotModifyCompleted)
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
// modules/todo/internal/application/command/create_todo_test.go
func TestCreateTodoCommand_Execute(t *testing.T) {
    // Mock only the PORTS (architectural boundaries)
    todoRepo := mocks.NewTodoRepository(t)
    eventDispatcher := mocks.NewEventDispatcher(t)
    uow := mocks.NewUnitOfWork(t)

    // Mock expectations
    uow.On("WithTransaction", ctx, mock.Anything).RunAndReturn(
        func(ctx context.Context, fn func(context.Context) error) error { return fn(ctx) },
    )
    todoRepo.On("Save", ctx, mock.Anything).Return(nil)
    eventDispatcher.On("Dispatch", ctx, mock.Anything).Return(nil)

    // Use REAL service — test actual business logic, not just wiring
    svc := application.NewTodoApplicationService(todoRepo, uow, eventDispatcher)
    resp, err := svc.CreateTodo(ctx, &todov1.CreateTodoRequest{
        Title:  "Buy milk",
        UserId: uuid.NewString(),
    })

    require.NoError(t, err)
    assert.NotEmpty(t, resp.GetTodo().GetId())
    todoRepo.AssertExpectations(t)
}
```

**Characteristics:**
- Fast (~1-10ms per test)
- Tests use case orchestration logic
- Mocked **ports only** (infrastructure boundaries)
- Tests error handling and edge cases

## ⚠️ The Mockist Trap: What NOT to Mock

**Common pitfall:** Generating mocks for every interface and using them in all tests leads to **fragile, implementation-coupled tests** that break on every refactoring.

### The Golden Rule: Mock Architectural Boundaries Only

**✅ DO Mock (Ports — Infrastructure Boundaries):**
- **Database access**: `TodoRepository`, `UnitOfWork`
- **External APIs**: `PaymentGateway`, `EmailService`
- **Message brokers**: `EventDispatcher`
- **Non-deterministic systems**: `Clock`, `IDGenerator`

These cross architectural boundaries into infrastructure. They're slow, have external state, or introduce non-determinism.

**❌ DON'T Mock (Internal Business Logic):**
- **Domain entities/value objects**: `Todo`, `User`, `Title`, `Status` — just instantiate them!
- **Application services**: `TodoService`, `AuthService` — use real implementations with mocked ports
- **In-memory operations**: Pure calculation services, formatters, validators
- **Standard library types**: `io.Reader`, `io.Writer` — use `bytes.Buffer` or `strings.Reader`

### Why Over-Mocking Breaks Tests

When you mock internal services, you test **implementation details** instead of **behavior**:

```go
// ❌ BAD: Testing implementation details (breaks on every internal refactor)
func TestHandler_CreateTodo_OverMocked(t *testing.T) {
    mockService := mocks.NewMockTodoService(t)
    mockService.EXPECT().
        CreateTodo(ctx, mock.Anything).
        Return(&response, nil)

    handler := NewTodoHandler(mockService)
    // You're only testing that handler calls service — no real logic tested!
}

// ✅ GOOD: Testing actual behavior through the real service
func TestHandler_CreateTodo_Behavioral(t *testing.T) {
    // Mock only PORTS (architectural boundaries)
    mockRepo := mocks.NewTodoRepository(t)
    mockUoW  := mocks.NewUnitOfWork(t)
    mockDisp := mocks.NewEventDispatcher(t)

    mockUoW.EXPECT().WithTransaction(ctx, mock.Anything).RunAndReturn(
        func(ctx context.Context, fn func(context.Context) error) error { return fn(ctx) },
    )
    mockRepo.EXPECT().Save(ctx, mock.Anything).Return(nil)
    mockDisp.EXPECT().Dispatch(ctx, mock.Anything).Return(nil)

    // Use REAL service — tests the actual business logic!
    svc     := application.NewTodoApplicationService(mockRepo, mockUoW, mockDisp)
    handler := connect.NewTodoHandler(svc)

    // Tests handler → service → ports flow with real domain logic
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
    r.todos[todo.ID().String()] = todo
    return nil
}

// Use in tests — behaves like a real database, no .EXPECT() needed!
```

**Benefits of Fakes:**
- Behave like real implementations
- No brittle `.EXPECT()` setup in every test
- Can be reused across test suites
- Useful for local development without Docker

### Summary: Mock Configuration

Your `mockery.yaml` should **ONLY** include infrastructure ports:

```yaml
# ✅ CORRECT — only mock architectural boundaries
packages:
  github.com/pivaldi/mmw-todo/internal/application/ports:
    interfaces:
      TodoRepository: {}   # Database boundary
      UnitOfWork: {}       # Transaction boundary
      EventDispatcher: {}  # Message broker boundary

# ❌ WRONG — don't add these:
#  github.com/pivaldi/mmw-todo/internal/application:
#    interfaces:
#      TodoService: {}     # This is internal application logic!
```

**Key insight:** Mocking is for *architectural boundaries*, not *code organization*. Mock what's slow, external, or non-deterministic. Use real implementations for everything else.

## 3. Integration Tests (Adapters)

**Focus:** Adapters with real external systems

```go
// modules/todo/internal/adapters/outbound/persistence/postgres/todo_repository_test.go
func TestPostgresTodoRepository_Save(t *testing.T) {
    if testing.Short() {
        t.Skip("Skipping integration test")
    }

    pool := testhelper.SetupPostgres(t) // testcontainers
    uow  := pfuow.New(pool)
    repo := postgres.NewPostgresTodoRepository(uow)

    title, _ := domain.NewTitle("Buy milk")
    todo, _   := domain.NewTodo(title, domain.EmptyDescription, nil, uuid.New())

    err := uow.WithTransaction(ctx, func(txCtx context.Context) error {
        return repo.Save(txCtx, todo)
    })
    require.NoError(t, err)

    found, err := repo.FindByID(ctx, todo.ID())
    require.NoError(t, err)
    assert.Equal(t, todo.Title(), found.Title())
}
```

**Characteristics:**
- Medium speed (~10-100ms per test)
- Real database via testcontainers
- Tests adapter implementation correctness
- Use `testing.Short()` to skip in fast test runs

## 4. Contract Tests (Service Boundary Verification)

**Focus:** Verify that a module correctly implements its own public API contract

**Important:** Contract tests live in the **PROVIDER** module, not the consumer. Each module tests its own implementation of its contract interface.

```go
// modules/todo/test/contract/contract_test.go
func TestTodoService_ContractImplementation(t *testing.T) {
    pool := testhelper.SetupPostgres(t)
    todo.Migrate(ctx, pool) // run migrations

    // Wire the real service with a real DB
    infra := todo.Infrastructure{
        DBPool:  pool,
        EventBus: pfevents.NewWatermillBus(rawBus),
        // ...
    }
    m, _ := todo.New(infra)

    // Test the service satisfies the defTodo.TodoService interface
    resp, err := m.Handler() // or call service directly

    // Validate DTO mapping (domain entity → proto response)
    assert.NotEmpty(t, resp.GetTodo().GetId())

    // Validate error contract
    _, err = svc.GetTodo(ctx, &todov1.GetTodoRequest{Id: "nonexistent"})
    assert.Contains(t, err.Error(), "not found")
}
```

**Characteristics:**
- Tests that the module correctly implements its contract interface
- Validates DTO mapping (domain entities → proto DTOs)
- Validates error translation (domain errors → platform errors → Connect codes)
- Lives in the **provider** module; consumers never import the provider's implementation

## 5. System Tests

**Focus:** Full user scenarios across modules

```go
// test/system/todo_flow_test.go
func TestTodoFlow_CreateAndComplete(t *testing.T) {
    env := testhelper.StartSystem(t) // testcontainers postgres + all modules

    // Register and login via auth module
    regResp, _ := env.AuthPublicClient.Register(ctx, &authv1.RegisterRequest{
        Email: "user@example.com", Password: "secret",
    })
    loginResp, _ := env.AuthPublicClient.Login(ctx, &authv1.LoginRequest{
        Email: "user@example.com", Password: "secret",
    })

    // Create a todo via the todo module (with bearer token)
    todoClient := env.TodoClientWithToken(loginResp.GetToken())
    createResp, err := todoClient.CreateTodo(ctx, &todov1.CreateTodoRequest{Title: "Buy milk"})
    require.NoError(t, err)

    // Complete the todo
    _, err = todoClient.CompleteTodo(ctx, &todov1.CompleteTodoRequest{Id: createResp.GetTodo().GetId()})
    require.NoError(t, err)

    // Verify final state
    getResp, _ := todoClient.GetTodo(ctx, &todov1.GetTodoRequest{Id: createResp.GetTodo().GetId()})
    assert.Equal(t, "COMPLETED", getResp.GetTodo().GetStatus())
}
```

**Characteristics:**
- Tests complete flows across module boundaries
- All modules running in the same process (or with separate HTTP servers via testcontainers)
- Real interactions: HTTP → Connect handler → application service → PostgreSQL → outbox → Watermill

## Running Tests

```bash
# Unit tests only (fast — skips integration)
go test ./... -short

# All tests (includes integration, requires Docker)
go test ./...

# With coverage
go test ./... -coverprofile=coverage.out
go tool cover -html=coverage.out

# Specific module
cd modules/todo && go test ./...

# System tests (from workspace root)
cd test/system && go test ./...

# Coverage table (via mmw-cli)
cd modules/todo && mmw test coverage --min 70

# Via mise
mise run test:unit
mise run test:integration
mise run test:all
```
