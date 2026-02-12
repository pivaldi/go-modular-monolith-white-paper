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
// services/authsvc/internal/domain/user/user_test.go
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
// services/authsvc/internal/application/command/login_test.go
func TestLoginCommand_Execute(t *testing.T) {
    // Setup mocks
    userRepo := mocks.NewUserRepository(t)
    sessionRepo := mocks.NewSessionRepository(t)
    authorClient := mocks.NewAuthorClient(t)

    // Mock expectations
    userRepo.On("FindByEmail", ctx, testEmail).Return(testUser, nil)
    sessionRepo.On("Save", ctx, mock.Anything).Return(nil)
    authorClient.On("GetAuthor", ctx, "user-123").Return(&ports.AuthorInfo{
        Name: "John Doe",
    }, nil)

    // Execute
    cmd := command.NewLoginCommand(userRepo, sessionRepo, authorClient, logger)
    result, err := cmd.Execute(ctx, command.LoginInput{
        Email:    "user@example.com",
        Password: "password123",
    })

    // Assert
    require.NoError(t, err)
    assert.NotEmpty(t, result.Token)
    assert.Equal(t, "John Doe", result.AuthorName)

    userRepo.AssertExpectations(t)
}
```

**Characteristics:**
- Fast (~1-10ms per test)
- Tests use case logic
- Mocked external dependencies
- Tests error handling and edge cases

## 3. Integration Tests (Adapters)

**Focus:** Adapters with real external systems

```go
// services/authsvc/internal/adapters/outbound/persistence/postgres/user_repository_test.go
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

## 4. Contract Tests (Bridge/Service Boundaries)

**Focus:** Verify that a service correctly implements its own public API contract

**Important:** Contract tests live in the **PROVIDER** service, not the consumer. Each service tests its own implementation of its bridge interface.

```go
// services/authorsvc/test/contract/bridge_test.go
func TestAuthorService_BridgeContract(t *testing.T) {
    // Setup: Initialize authorsvc with test data
    // This is authorsvc testing its OWN implementation
    app := setupTestAuthorService(t)
    defer app.Cleanup()

    // Create InprocServer (the bridge implementation authorsvc provides)
    server := inbound_bridge.NewInprocServer(
        app.GetAuthorQuery,
        app.CreateAuthorCommand,
        // ... other dependencies
    )

    // Test that it correctly implements bridge.AuthorService interface
    author, err := server.GetAuthor(context.Background(), "test-author-123")
    require.NoError(t, err)
    assert.Equal(t, "Test Author", author.Name)

    // Test error contract
    _, err = server.GetAuthor(context.Background(), "nonexistent")
    assert.ErrorIs(t, err, bridge_authorsvc.ErrAuthorNotFound)

    // Test DTO mapping (domain entity → bridge DTO)
    assert.NotEmpty(t, author.ID)
    assert.NotEmpty(t, author.CreatedAt)
}
```

**Characteristics:**
- Tests that the service correctly implements its bridge interface
- Validates DTO mapping (domain entities → bridge DTOs)
- Validates error translation (domain errors → bridge errors)
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

    // Create author profile
    authorResp, err := env.AuthorClient.CreateAuthor(ctx, "Jane Smith", "Writer")
    require.NoError(t, err)

    // Login
    loginResp, err := env.AuthClient.Login(ctx, "user@example.com", "SecurePass123!")
    require.NoError(t, err)
    assert.Equal(t, "Jane Smith", loginResp.AuthorName)
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
cd services/authsvc
go test ./...

# Integration tests only
go test ./... -run Integration

# Via mise
mise run test:unit
mise run test:integration
mise run test:all
```
