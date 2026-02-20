# Technical Implementation Reference

Go Workspaces Modular Monolith with Contract Definitions - Pure Technical Documentation

## 1. Workspace Architecture

### Go Workspace Configuration

**File:** `go.work`

```
use (
    ./contracts
    ./contracts/definitions/serviceasvc
    ./contracts/definitions/servicebsvc
    ./services/serviceasvc
    ./services/servicebsvc
    ./test/e2e
)
```

**Purpose:** Coordinates multiple independent Go modules in a single repository.

**Key Properties:**
- Each `use` entry is an independent Go module with its own `go.mod`
- Workspace makes cross-module development seamless (no publishing required)
- `go test ./...` works across entire workspace
- IDE jump-to-definition works across modules

### Module Organization

```
service-manager/
├── go.work                           # Workspace coordinator
├── contracts/                        # Unified contracts module
│   ├── go.mod                        # Module: contracts
│   ├── definitions/                  # Contract definitions (pure interfaces)
│   │   ├── serviceasvc/              # Service A public contract
│   │   │   ├── go.mod                # ZERO dependencies
│   │   │   ├── api.go                # Interface definition
│   │   │   ├── dto.go                # Data transfer objects
│   │   │   ├── errors.go             # Public error types
│   │   │   └── inproc_client.go      # In-process client wrapper
│   │   └── servicebsvc/              # Service B public contract
│   ├── proto/                        # Protobuf schemas (optional)
│   │   ├── buf.yaml
│   │   ├── servicea/v1/servicea.proto
│   │   └── serviceb/v1/serviceb.proto
│   └── gen/                          # Generated code from protobuf
│       ├── go/servicea/v1/
│       └── ts/servicea/v1/
├── services/
│   ├── serviceasvc/                  # Service A
│   │   ├── go.mod                    # Module: services/serviceasvc
│   │   │                             # Dependencies: contracts/definitions/serviceasvc
│   │   ├── cmd/serviceasvc/main.go   # Service entry point
│   │   └── internal/                 # Private implementation
│   └── servicebsvc/                  # Service B
│       ├── go.mod                    # Module: services/servicebsvc
│       │                             # Dependencies: contracts/definitions/serviceasvc
│       └── internal/
└── test/e2e/                         # End-to-end tests
    └── go.mod                        # Module: test/e2e
```

### Module Dependency Rules

**Enforced by Go compiler:**

1. **Contract Definitions:** Zero dependencies
   ```go
   // contracts/definitions/serviceasvc/go.mod
   module github.com/example/service-manager/contracts/definitions/serviceasvc
   go 1.21
   // NO require statements
   ```

2. **Services:** Can import contract definitions only
   ```go
   // services/servicebsvc/go.mod
   module github.com/example/service-manager/services/servicebsvc
   require (
       github.com/.../contracts/definitions/serviceasvc v1.0.0
   )
   // Cannot import services/serviceasvc/internal (compiler error)
   ```

3. **Internal Packages:** Cannot be imported across module boundaries
   - `services/serviceasvc/internal/` is inaccessible to `servicebsvc`
   - Enforced by Go's internal package rules + separate modules

## 2. Contract Definition Layer

### Contract Structure

**Location:** `contracts/definitions/serviceasvc/`

**Purpose:** Public API contract that **defines** in-process communication with module independence.

### Contract Components

**1. Interface Definition** (`api.go`)

```go
package serviceasvc

import "context"

// ServiceA defines the public API contract
type ServiceA interface {
    GetEntityA(ctx context.Context, id string) (*ServiceADTO, error)
    CreateEntityA(ctx context.Context, req *CreateEntityARequest) (*ServiceADTO, error)
    UpdateEntityA(ctx context.Context, id string, req *UpdateEntityARequest) (*ServiceADTO, error)
    ListEntityA(ctx context.Context, filter *ListEntityAFilter) ([]*ServiceADTO, error)
}
```

**2. Data Transfer Objects** (`dto.go`)

```go
package serviceasvc

type ServiceADTO struct {
    ID        string
    Name      string
    Bio       string
    AvatarURL string
}

type CreateEntityARequest struct {
    Name string
    Bio  string
}

type UpdateEntityARequest struct {
    Name *string  // Optional fields use pointers
    Bio  *string
}

type ListEntityAFilter struct {
    Limit  int
    Offset int
}
```

**3. Error Types** (`errors.go`)

```go
package serviceasvc

import "errors"

var (
    ErrEntityANotFound     = errors.New("entity not found")
    ErrInvalidEntityAData  = errors.New("invalid entity data")
    ErrServiceAUnavailable = errors.New("service unavailable")
)
```

**4. In-Process Client** (`inproc_client.go`)

```go
package serviceasvc

import "context"

// InprocClient wraps any ServiceA implementation for in-process calls
type InprocClient struct {
    server ServiceA  // Interface, not concrete type
}

func NewInprocClient(server ServiceA) *InprocClient {
    return &InprocClient{server: server}
}

func (c *InprocClient) GetEntityA(ctx context.Context, id string) (*ServiceADTO, error) {
    return c.server.GetEntityA(ctx, id)
}

func (c *InprocClient) CreateEntityA(ctx context.Context, req *CreateEntityARequest) (*ServiceADTO, error) {
    return c.server.CreateEntityA(ctx, req)
}

// ... other methods delegate to server interface
```

### Contract Dependency Rules

**What contracts contain:**
- Interface definitions
- DTOs (pure data structures, no methods with business logic)
- Error constants
- InprocClient (thin wrapper)

**What contracts NEVER contain:**
- Business logic or validation
- External dependencies (`go.mod` has zero `require` statements)
- Imports of `internal/` packages
- Helper functions or utilities
- Global state or caches

**Enforcement:**
- `arch-test` tool validates zero dependencies
- `arch-test` validates no `internal/` imports
- Code review catches logical coupling

## 3. Service Module Structure

### Directory Layout

```
services/serviceasvc/
├── go.mod                            # Independent module
├── cmd/
│   └── serviceasvc/
│       └── main.go                   # Service entry point
├── internal/                         # Private implementation
│   ├── domain/                       # Domain layer (entities, value objects)
│   ├── application/                  # Application layer (use cases, ports)
│   ├── adapters/                     # Adapters layer (I/O boundaries)
│   │   ├── inbound/                  # Inbound adapters
│   │   │   ├── contracts/            # Contract adapter (InprocServer)
│   │   │   │   └── inproc_server.go
│   │   │   ├── http/                 # HTTP REST adapter
│   │   │   └── connect/              # gRPC/Connect adapter
│   │   └── outbound/                 # Outbound adapters
│   │       ├── persistence/postgres/ # Database adapter
│   │       └── serviceaclient/       # Client adapters
│   └── infra/                        # Infrastructure (config, DB, etc.)
├── test/
│   └── contract/                     # Contract tests (provider side)
└── README.md
```

### go.mod Configuration

```go
module github.com/example/service-manager/services/serviceasvc

go 1.21

require (
    // Contract definition this service implements
    github.com/.../contracts/definitions/serviceasvc v1.0.0

    // Standard dependencies
    github.com/lib/pq v1.10.9
    golang.org/x/sync v0.3.0
)
```

### Internal Package Organization

**Domain Layer:** `internal/domain/`
- Pure business logic
- No external dependencies
- Aggregate structure: `domain/{aggregate}/`

**Application Layer:** `internal/application/`
- Use cases and orchestration
- Structure: `application/{command,query,ports}/`

**Adapters Layer:** `internal/adapters/`
- I/O boundaries
- Structure: `adapters/{inbound,outbound}/`

**Infrastructure Layer:** `internal/infra/`
- Technical concerns (config, database, observability)

## 4. Hexagonal Architecture Layers

### Layer Dependencies

```
Domain (pure business logic)
    ↑
Application (use cases, ports)
    ↑
Adapters (implementations)
    ↑
Infrastructure (wiring)
    ↑
main.go (composition root)
```

**Rule:** Dependencies point inward. Inner layers never depend on outer layers.

### Domain Layer

**Location:** `services/*/internal/domain/`

**Components:**

**Entities** (aggregate roots)
```go
// domain/user/user.go
type User struct {
    id             string
    email          Email
    hashedPassword HashedPassword
    createdAt      time.Time
    updatedAt      time.Time
}

func NewUser(id string, email Email, hashedPassword HashedPassword) *User {
    return &User{
        id:             id,
        email:          email,
        hashedPassword: hashedPassword,
        createdAt:      time.Now(),
        updatedAt:      time.Now(),
    }
}

func (u *User) ChangePassword(oldPassword string, newPassword string) error {
    // Verify old password
    if !u.hashedPassword.Verify(oldPassword) {
        return ErrInvalidPassword
    }

    // Business rule: Validate new password strength
    if err := ValidatePasswordStrength(newPassword); err != nil {
        return err
    }

    // Business rule: New password cannot be same as old
    if u.hashedPassword.Verify(newPassword) {
        return ErrPasswordMustBeDifferent
    }

    // Hash new password
    newHash, err := HashPassword(newPassword)
    if err != nil {
        return err
    }

    // Update entity state (persistence happens via repository.Save() in application layer)
    u.hashedPassword = newHash
    u.updatedAt = time.Now()

    return nil
}

// VerifyPassword checks if the given password matches
func (u *User) VerifyPassword(password string) bool {
    return u.hashedPassword.Verify(password)
}

// Getters
func (u *User) ID() string { return u.id }
func (u *User) Email() Email { return u.email }
func (u *User) CreatedAt() time.Time { return u.createdAt }
func (u *User) UpdatedAt() time.Time { return u.updatedAt }
```

**Value Objects**
```go
// domain/user/email.go
type Email struct {
    value string
}

func NewEmail(email string) (Email, error) {
    email = strings.TrimSpace(strings.ToLower(email))
    if !emailRegex.MatchString(email) {
        return Email{}, errors.New("invalid email format")
    }
    return Email{value: email}, nil
}

func (e Email) String() string { return e.value }
```

**Repository Interfaces** (ports owned by domain)
```go
// domain/user/repository.go
type Repository interface {
    Save(ctx context.Context, user *User) error
    FindByID(ctx context.Context, id string) (*User, error)
    FindByEmail(ctx context.Context, email Email) (*User, error)
}
```

**Properties:**
- No external dependencies (standard library only)
- Fully testable without infrastructure
- Encapsulates business rules

### Application Layer

**Location:** `services/*/internal/application/`

**Structure:**
- `command/` - Write operations
- `query/` - Read operations
- `ports/` - Interface definitions for adapters
- `dto/` - Application data transfer objects

**Command Example**
```go
// application/command/login.go
package command

import (
    "context"
    "errors"
    "time"

    "github.com/example/service-manager/services/servicebsvc/internal/application/ports"
    "github.com/example/service-manager/services/servicebsvc/internal/domain/session"
    "github.com/example/service-manager/services/servicebsvc/internal/domain/user"
)

type LoginCommand struct {
    userRepo       user.Repository      // Domain interface
    sessionRepo    session.Repository   // Domain interface
    authorClient   ports.AuthorClient   // Application port
    logger         ports.Logger         // Application port
}

func NewLoginCommand(
    userRepo user.Repository,
    sessionRepo session.Repository,
    authorClient ports.AuthorClient,
    logger ports.Logger,
) *LoginCommand {
    return &LoginCommand{
        userRepo:     userRepo,
        sessionRepo:  sessionRepo,
        authorClient: authorClient,
        logger:       logger,
    }
}

type LoginInput struct {
    Email    string
    Password string
}

type LoginOutput struct {
    Token      string
    UserID     string
    AuthorName string
    ExpiresAt  time.Time
}

func (c *LoginCommand) Execute(ctx context.Context, input LoginInput) (*LoginOutput, error) {
    // 1. Validate with domain
    email, err := user.NewEmail(input.Email)
    if err != nil {
        return nil, ErrInvalidCredentials
    }

    // 2. Fetch entity
    u, err := c.userRepo.FindByEmail(ctx, email)
    if err != nil {
        return nil, err
    }

    // 3. Domain logic
    if !u.VerifyPassword(input.Password) {
        return nil, ErrInvalidCredentials
    }

    // 4. Create session
    sess := session.New(u.ID(), 24*time.Hour)

    // 5. Call external service (via port)
    authorInfo, err := c.authorClient.GetAuthor(ctx, u.ID())
    // Handle error with graceful degradation

    // 6. Persist
    if err := c.sessionRepo.Save(ctx, sess); err != nil {
        return nil, err
    }

    // 7. Return output
    return &LoginOutput{
        Token:      sess.Token().String(),
        UserID:     u.ID(),
        AuthorName: authorInfo.Name,
        ExpiresAt:  sess.ExpiresAt(),
    }, nil
}
```

**Port Definition**
```go
// application/ports/author_client.go
type AuthorClient interface {
    GetAuthor(ctx context.Context, authorID string) (*AuthorInfo, error)
}

type AuthorInfo struct {
    ID        string
    Name      string
    Bio       string
    AvatarURL string
}
```

**Properties:**
- Orchestrates domain objects
- Defines ports (interfaces) for external dependencies
- No knowledge of HTTP, databases, or frameworks
- Transaction boundaries

### Adapters Layer

**Location:** `services/*/internal/adapters/`

#### Inbound Adapters

**Contract Adapter** (`adapters/inbound/contracts/inproc_server.go`)

```go
package contracts

import (
    "context"
    "github.com/example/.../contracts/definitions/serviceasvc"
    "github.com/example/.../services/serviceasvc/internal/application/command"
    "github.com/example/.../services/serviceasvc/internal/application/query"
)

// InprocServer implements the contract interface
type InprocServer struct {
    getEntityAQuery  *query.GetEntityAQuery
    createEntityACmd *command.CreateEntityACommand
}

// NewInprocServer returns the interface type (loose coupling)
func NewInprocServer(
    getEntityAQuery *query.GetEntityAQuery,
    createEntityACmd *command.CreateEntityACommand,
) serviceasvc.ServiceA {
    return &InprocServer{
        getEntityAQuery:  getEntityAQuery,
        createEntityACmd: createEntityACmd,
    }
}

func (s *InprocServer) GetEntityA(ctx context.Context, id string) (*serviceasvc.ServiceADTO, error) {
    // 1. Call application layer
    result, err := s.getEntityAQuery.Execute(ctx, query.GetEntityAInput{ID: id})
    if err != nil {
        // 2. Map domain errors to contract errors
        if errors.Is(err, domain.ErrEntityANotFound) {
            return nil, serviceasvc.ErrEntityANotFound
        }
        return nil, serviceasvc.ErrServiceAUnavailable
    }

    // 3. Map domain entity to contract DTO
    return &serviceasvc.ServiceADTO{
        ID:        result.Entity.ID(),
        Name:      result.Entity.Name().String(),
        Bio:       result.Entity.Bio(),
        AvatarURL: result.Entity.AvatarURL(),
    }, nil
}
```

**HTTP Adapter** (`adapters/inbound/http/handlers/login.go`)

```go
package handlers

import (
    "encoding/json"
    "net/http"
    "github.com/example/.../internal/application/command"
)

type LoginHandler struct {
    loginCmd *command.LoginCommand
}

type LoginRequest struct {
    Email    string `json:"email"`
    Password string `json:"password"`
}

type LoginResponse struct {
    Token     string `json:"token"`
    UserID    string `json:"user_id"`
    ExpiresAt int64  `json:"expires_at"`
}

func (h *LoginHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // 1. Parse HTTP request
    var req LoginRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "invalid request", http.StatusBadRequest)
        return
    }

    // 2. Call application layer
    output, err := h.loginCmd.Execute(r.Context(), command.LoginInput{
        Email:    req.Email,
        Password: req.Password,
    })
    if err != nil {
        // 3. Map errors to HTTP status codes
        switch {
        case errors.Is(err, command.ErrInvalidCredentials):
            http.Error(w, "invalid credentials", http.StatusUnauthorized)
        default:
            http.Error(w, "internal error", http.StatusInternalServerError)
        }
        return
    }

    // 4. Return HTTP response
    resp := LoginResponse{
        Token:     output.Token,
        UserID:    output.UserID,
        ExpiresAt: output.ExpiresAt.Unix(),
    }
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(resp)
}
```

#### Outbound Adapters

**Database Adapter** (`adapters/outbound/persistence/postgres/user_repository.go`)

```go
package postgres

import (
    "context"
    "database/sql"
    "github.com/example/.../internal/domain/user"
)

type UserRepository struct {
    db *sql.DB
}

func NewUserRepository(db *sql.DB) *UserRepository {
    return &UserRepository{db: db}
}

func (r *UserRepository) Save(ctx context.Context, u *user.User) error {
    query := `
        INSERT INTO users (id, email, hashed_password, created_at, updated_at)
        VALUES ($1, $2, $3, $4, $5)
        ON CONFLICT (id) DO UPDATE SET
            email = EXCLUDED.email,
            hashed_password = EXCLUDED.hashed_password,
            updated_at = EXCLUDED.updated_at
    `
    _, err := r.db.ExecContext(ctx, query,
        u.ID(),
        u.Email().String(),
        u.HashedPassword().String(),
        u.CreatedAt(),
        u.UpdatedAt(),
    )
    return err
}

func (r *UserRepository) FindByEmail(ctx context.Context, email user.Email) (*user.User, error) {
    query := `SELECT id, email, hashed_password, created_at, updated_at FROM users WHERE email = $1`

    var id, emailStr, passwordStr string
    var createdAt, updatedAt time.Time

    err := r.db.QueryRowContext(ctx, query, email.String()).Scan(&id, &emailStr, &passwordStr, &createdAt, &updatedAt)
    if errors.Is(err, sql.ErrNoRows) {
        return nil, user.ErrUserNotFound
    }
    if err != nil {
        return nil, err
    }

    // Reconstruct domain entity
    e, _ := user.NewEmail(emailStr)
    p := user.NewHashedPassword(passwordStr)
    return user.Reconstruct(id, e, p, createdAt, updatedAt), nil
}
```

**Service Client Adapter** (`adapters/outbound/authorclient/inproc/client.go`)

```go
package inproc

import (
    "context"
    "github.com/example/.../contracts/definitions/authorservice"
    "github.com/example/.../internal/application/ports"
)

// Client implements ports.AuthorClient using the contract definition
type Client struct {
    contract authorservice.AuthorService
}

func NewClient(contract authorservice.AuthorService) *Client {
    return &Client{contract: contract}
}

func (c *Client) GetAuthor(ctx context.Context, authorID string) (*ports.AuthorInfo, error) {
    // Call contract
    dto, err := c.contract.GetAuthor(ctx, authorID)
    if err != nil {
        // Map contract errors to application errors
        if errors.Is(err, authorservice.ErrAuthorNotFound) {
            return nil, ports.ErrAuthorNotFound
        }
        return nil, ports.ErrAuthorServiceDown
    }

    // Map contract DTO to application DTO
    return &ports.AuthorInfo{
        ID:        dto.ID,
        Name:      dto.Name,
        Bio:       dto.Bio,
        AvatarURL: dto.AvatarURL,
    }, nil
}
```

### Infrastructure Layer

**Location:** `services/*/internal/infra/`

**Configuration** (`infra/config/config.go`)

```go
package config

import (
    "embed"
    "gopkg.in/yaml.v3"
)

//go:embed defaults.yaml
var defaultsFS embed.FS

type Config struct {
    HTTPPort string     `yaml:"http_port"`
    DB       DBConfig   `yaml:"db"`
    Logger   LogConfig  `yaml:"logger"`
}

func Load() (*Config, error) {
    // Load embedded defaults
    data, _ := defaultsFS.ReadFile("defaults.yaml")
    var cfg Config
    yaml.Unmarshal(data, &cfg)

    // Apply environment overrides
    if port := os.Getenv("SERVICEA_HTTP_PORT"); port != "" {
        cfg.HTTPPort = port
    }

    return &cfg, nil
}
```

## 5. Runtime Orchestration

### Composition Root (main.go)

**Location:** `cmd/monolith/main.go` or service-specific `cmd/serviceasvc/main.go`

**Structure:**

```go
package main

import (
    "context"
    "log"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"

    "golang.org/x/sync/errgroup"

    // Contract definitions
    authorservice "github.com/.../contracts/definitions/authorservice"

    // Service configs
    authorconfig "github.com/.../services/authorservice/config"
    authconfig "github.com/.../services/authservice/config"

    // Service adapters
    authoradapters "github.com/.../services/authorservice/internal/adapters/inbound/contracts"
    authorhttp "github.com/.../services/authorservice/internal/adapters/inbound/http"
    authadapters "github.com/.../services/authservice"
    authhttp "github.com/.../services/authservice/internal/adapters/inbound/http"
)

func main() {
    // 1. Context with signal handling
    ctx, cancel := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
    defer cancel()

    // 2. Load configurations
    authorCfg, _ := authorconfig.Load()
    authCfg, _ := authconfig.Load()

    // 3. Initialize provider service
    authorServer := authoradapters.NewInprocServer(authorCfg.DB, authorCfg.Logger)

    // 4. Wrap with contract client
    authorClient := authorservice.NewInprocClient(authorServer)

    // 5. Initialize consumer service
    authService := authadapters.New(authCfg, authorClient)

    // 6. Create HTTP handlers
    authorHTTPHandler := authorhttp.NewHandler(authorCfg)
    authHTTPHandler := authhttp.NewHandler(authCfg, authService)

    // 7. Run services via errgroup (shared fate supervision)
    // Note: See "Worker Service Pattern" section for non-HTTP services
    g, gCtx := errgroup.WithContext(ctx)

    // Start Author Service HTTP server
    g.Go(func() error {
        server := &http.Server{
            Addr:    authorCfg.HTTPPort,
            Handler: authorHTTPHandler,
        }

        // Graceful shutdown watcher
        go func() {
            <-gCtx.Done()  // Fires when ANY service fails
            log.Println("Shutting down Author Service...")
            shutdownCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
            defer cancel()
            server.Shutdown(shutdownCtx)
        }()

        log.Printf("Author Service listening on %s", authorCfg.HTTPPort)
        if err := server.ListenAndServe(); err != http.ErrServerClosed {
            return err  // Error triggers shutdown of ALL services
        }

        return nil
    })

    // Start Auth Service HTTP server
    g.Go(func() error {
        server := &http.Server{
            Addr:    authCfg.HTTPPort,
            Handler: authHTTPHandler,
        }

        // Graceful shutdown watcher
        go func() {
            <-gCtx.Done()  // Fires when ANY service fails
            log.Println("Shutting down Auth Service...")
            shutdownCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
            defer cancel()
            server.Shutdown(shutdownCtx)
        }()

        log.Printf("Auth Service listening on %s", authCfg.HTTPPort)
        if err := server.ListenAndServe(); err != http.ErrServerClosed {
            return err  // Error triggers shutdown of ALL services
        }

        return nil
    })

    // 7. Wait for all services (blocks until error or signal)
    if err := g.Wait(); err != nil {
        log.Fatalf("Monolith shutdown: %v", err)
    }

    log.Println("Monolith shutdown gracefully")
}
```

**Wiring Flow:**

1. `authorServer := NewInprocServer(...)` - Creates server (returns `AuthorService` interface)
2. `authorClient := NewInprocClient(authorServer)` - Wraps server in client
3. `authService := New(cfg, authorClient)` - Injects client into consumer
4. Services run via `errgroup` (shared fate supervision)

### Supervision with errgroup

**Pattern:** Shared fate architecture

**Definition:** All services share the same fate - if one fails, all shut down together and the process exits.

Concrete example when Service A fails:

- **WITHOUT shared fate**
  - Service A: Crashed (for exemple database connection fails)
  - Service B: Running (but calls to Service A fail)
  - Service C: Running (partially functional)
  - Result: Zombie monolith in undefined state
- **WITH shared fate (errgroup)**
  - Service A: Returns error
  - errgroup: Cancels context for all services
  - Service B: Detects gCtx.Done() → graceful shutdown
  - Service C: Detects gCtx.Done() → graceful shutdown
  - Process: Exits
  - Kubernetes: Restarts entire pod with clean state

**Implementation:**

```go
g, gCtx := errgroup.WithContext(ctx)

// Start Service A
g.Go(func() error {
    server := &http.Server{Addr: ":8081", Handler: handler}

    // Graceful shutdown watcher
    go func() {
        <-gCtx.Done()  // Fires when ANY service fails or signal received
        shutdownCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
        defer cancel()
        server.Shutdown(shutdownCtx)
    }()

    // Blocking service until shutdown
    if err := server.ListenAndServe(); err != http.ErrServerClosed {
        return err  // Error propagates to errgroup → cancels gCtx
    }

    return nil
})

// Start Service B (same pattern)
g.Go(func() error {
    // ... similar structure ...
})

// Wait for all services (blocks until error or signal)
if err := g.Wait(); err != nil {
    log.Fatal(err)  // Exit process → orchestrator restarts
}
```

**Why this approach:**
- **Clean state:** Either all services healthy or none running (no partial failures)
- **Simpler:** No complex health checks or partial failure handling
- **Reliable:** Orchestrator (Kubernetes/Docker/SystemD) detects exit and restarts fresh pod
- **Prevents corruption:** Sick service doesn't partially corrupt shared resources

**Alternative:** For fine-grained fault tolerance, use [github.com/thejerf/suture](https://github.com/thejerf/suture) for Erlang-style supervision trees.

### Worker Service Pattern (Non-Blocking Services)

Services without HTTP servers (background workers, message consumers, cache warmers) need a different structure. They initialize resources, then block until shutdown.

**Worker Service Structure:**

```go
// services/workerservice/internal/worker/worker.go
package worker

import (
    "context"
    "database/sql"
    "fmt"
    "log"
    "time"
)

type Worker struct {
    db     *sql.DB
    config *Config
}

func New(config *Config) *Worker {
    return &Worker{config: config}
}

// Run initializes resources and blocks until context is cancelled
func (w *Worker) Run(ctx context.Context) error {
    // 1. Initialize database connection
    db, err := sql.Open("postgres", w.config.DatabaseURL)
    if err != nil {
        return fmt.Errorf("failed to connect to database: %w", err)
    }
    w.db = db
    defer db.Close()

    // 2. Verify connection is ready
    if err := db.PingContext(ctx); err != nil {
        return fmt.Errorf("database ping failed: %w", err)
    }

    log.Println("Worker service started (database connected)")

    // 3. Optional: Start background jobs
    go w.processQueue(ctx)

    // 4. Block until context is cancelled
    <-ctx.Done()

    // 5. Cleanup on shutdown
    log.Println("Worker service shutting down...")
    shutdownCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    // Wait for in-flight work to complete
    // Close connections, flush buffers, etc.
    if err := db.Close(); err != nil {
        log.Printf("Error closing database: %v", err)
    }

    log.Println("Worker service shutdown complete")
    return nil
}

func (w *Worker) processQueue(ctx context.Context) {
    ticker := time.NewTicker(10 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            // Process work from queue
            w.doWork(ctx)
        }
    }
}

func (w *Worker) doWork(ctx context.Context) {
    // Business logic here
}
```

**Using Worker in main.go:**

```go
// cmd/monolith/main.go

// Initialize worker service
workerService := worker.New(workerCfg)

// Start via errgroup (same as HTTP services)
g.Go(func() error {
    return workerService.Run(gCtx)
})
```

**Key Points:**

1. **`Run(ctx)` method:** Provides standard interface like HTTP servers
2. **Initialize resources:** Database connections, message queue clients, etc.
3. **Block on `<-ctx.Done()`:** Keeps goroutine alive until shutdown
4. **Graceful cleanup:** Close connections, wait for in-flight work
5. **main.go stays clean:** Just calls `workerService.Run(gCtx)`

**Pattern works for:**
- Background job processors
- Message queue consumers (RabbitMQ, Kafka)
- Cache warmers
- Database connection pools
- Any service needing initialization without HTTP

## 6. Protobuf Integration

### When to Use Protobuf

Use protobuf when services need network transport. Skip for in-process only.

### Directory Structure

```
contracts/
├── proto/                  # Source of truth (clean)
│   ├── buf.yaml
│   └── servicea/v1/
│       └── servicea.proto
└── gen/                    # Generated code (do not edit)
    ├── go/servicea/v1/
    │   ├── servicea.pb.go
    │   └── serviceav1connect/servicea.connect.go
    └── ts/servicea/v1/
```

### Protobuf Definition

**File:** `contracts/proto/servicea/v1/servicea.proto`

```protobuf
syntax = "proto3";
package servicea.v1;
option go_package = "github.com/.../contracts/gen/go/servicea/v1;serviceav1";

message EntityA {
  string id = 1;
  string name = 2;
  string bio = 3;
  string avatar_url = 4;
}

service ServiceAService {
  rpc GetEntityA(GetEntityARequest) returns (GetEntityAResponse);
  rpc CreateEntityA(CreateEntityARequest) returns (CreateEntityAResponse);
}
```

### Code Generation

**File:** `contracts/buf.gen.yaml`

```yaml
version: v1
plugins:
  - plugin: go
    out: gen/go
    opt: paths=source_relative
  - plugin: connect-go
    out: gen/go
    opt: paths=source_relative
```

**Commands:**

```bash
# Generate code from proto
cd contracts
buf generate

# Lint proto files
buf lint

# Check for breaking changes
buf breaking --against '.git#branch=main'
```

### Using Generated Code

**Contract Definition with Protobuf:**

```go
// contracts/definitions/serviceasvc/dto.go
import serviceav1 "github.com/.../contracts/gen/go/servicea/v1"

// Use generated protobuf types
type ServiceADTO = serviceav1.EntityA
```

**Network Client:**

```go
// adapters/outbound/authorclient/connect/client.go
import (
    serviceav1connect "github.com/.../contracts/gen/go/servicea/v1/serviceav1connect"
    "connectrpc.com/connect"
)

type Client struct {
    client serviceav1connect.ServiceAServiceClient
}

func NewClient(baseURL string) *Client {
    httpClient := &http.Client{Timeout: 5 * time.Second}
    client := serviceav1connect.NewServiceAServiceClient(httpClient, baseURL)
    return &Client{client: client}
}

func (c *Client) GetAuthor(ctx context.Context, id string) (*ports.AuthorInfo, error) {
    req := connect.NewRequest(&serviceav1.GetEntityARequest{Id: id})
    resp, err := c.client.GetEntityA(ctx, req)
    // Handle response
}
```

## 7. Testing & Operations

### Test Organization

**Unit Tests:** Co-located with code (`*_test.go`)
- Location: `internal/domain/**/*_test.go`, `internal/application/**/*_test.go`
- Purpose: Test business logic in isolation
- Dependencies: None (pure functions)

**Integration Tests:** Co-located with adapters
- Location: `internal/adapters/**/*_test.go`
- Purpose: Test adapters with real infrastructure
- Dependencies: Real database, cache, services

**Contract Tests:** Per-service provider tests
- Location: `services/*/test/contract/`
- Purpose: Verify service implements its contract correctly
- Provider tests its own contract, not consumer

**E2E Tests:** Root-level cross-service tests
- Location: `test/e2e/`
- Purpose: Complete user journeys across all services
- Dependencies: Full system running with services' composition

### Operational Commands

```bash
# Run full workspace tests
go test ./...

# Build all services
go build ./services/...

# Run specific service
go run ./services/serviceasvc/cmd/serviceasvc

# Verify module dependencies
go mod verify

# Update all dependencies
go get -u ./...

# Tidy all dependencies
go mod tidy
```

### Architecture Validation

**Tool:** `arch-test` (optional, in `tools/arch-test/`)

```bash
# Validate architectural boundaries
go run ./tools/arch-test

# Checks:
# - Contract definitions have zero dependencies
# - No service imports another service's internal/
# - Dependency flow rules (domain <- application <- adapters)
```

### Configuration

Each service loads config via:

```go
cfg, err := config.Load()
```

**Pattern:** Embedded defaults + environment overrides

**Environment Variables:**

```bash
# Service A
SERVICEA_HTTP_PORT=:8081
SERVICEA_DB_HOST=localhost
SERVICEA_DB_PASSWORD=secret

# Service B
SERVICEB_HTTP_PORT=:8082
SERVICEB_DB_HOST=localhost
```

### Deployment

**Monolith Mode:**
- Build single binary: `go build ./cmd/monolith`
- Deploy as single container
- All services run in one process

**Distributed Mode:**
- Build separate binaries: `go build ./services/serviceasvc/cmd/serviceasvc`
- Deploy as separate containers
- Swap in-process clients for network clients in main.go

### Key Files Reference

| File | Purpose |
|------|---------|
| `go.work` | Workspace coordination |
| `contracts/definitions/*/api.go` | Contract interface |
| `contracts/definitions/*/inproc_client.go` | In-process client wrapper |
| `services/*/internal/adapters/inbound/contracts/inproc_server.go` | Contract implementation |
| `services/*/internal/domain/` | Business logic |
| `services/*/internal/application/` | Use cases |
| `services/*/internal/adapters/` | I/O boundaries |
| `cmd/monolith/main.go` | Composition root |
| `contracts/proto/` | Protobuf schemas |
| `contracts/buf.gen.yaml` | Code generation config |

---

**End of Technical Reference**
