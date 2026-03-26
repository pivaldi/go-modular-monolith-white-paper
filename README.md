# Technical Implementation Reference

Go Workspaces Modular Monolith with Contract Definitions - Pure Technical Documentation.

**⚠️ Important Note:** This document provides technical implementation details. For architectural context, design rationale, and comparison with alternative approaches, see [white-paper.md](white-paper.md).

<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-refresh-toc -->
**Table of Contents**

- [Technical Implementation Reference](#technical-implementation-reference)
  - [1. Workspace Architecture](#1-workspace-architecture)
    - [Go Workspace Configuration](#go-workspace-configuration)
    - [Module Organization](#module-organization)
    - [Module Dependency Rules](#module-dependency-rules)
  - [2. Contract Definition Layer](#2-contract-definition-layer)
    - [Contract Structure](#contract-structure)
    - [Contract Components](#contract-components)
    - [Contract Dependency Rules](#contract-dependency-rules)
  - [3. Module Structure](#3-module-structure)
    - [Directory Layout](#directory-layout)
    - [go.mod Configuration](#gomod-configuration)
    - [Internal Package Organization](#internal-package-organization)
  - [4. Hexagonal Architecture Layers](#4-hexagonal-architecture-layers)
    - [Layer Dependencies](#layer-dependencies)
    - [Domain Layer](#domain-layer)
    - [Application Layer](#application-layer)
    - [Adapters Layer](#adapters-layer)
      - [Inbound Adapters](#inbound-adapters)
      - [Outbound Adapters](#outbound-adapters)
    - [Infrastructure Layer](#infrastructure-layer)
  - [5. Runtime Orchestration](#5-runtime-orchestration)
    - [mmw-platform: Module Lifecycle](#mmw-platform-module-lifecycle)
    - [Composition Root (main.go)](#composition-root-maingo)
    - [Supervision with errgroup](#supervision-with-errgroup)
  - [6. Protobuf Integration](#6-protobuf-integration)
    - [When to Use Protobuf](#when-to-use-protobuf)
    - [Directory Structure](#directory-structure)
    - [Protobuf Definition](#protobuf-definition)
    - [Code Generation](#code-generation)
    - [Using Generated Code](#using-generated-code)
  - [7. Testing & Operations](#7-testing--operations)
    - [Test Organization](#test-organization)
    - [Operational Commands](#operational-commands)
    - [Architecture Validation](#architecture-validation)
  - [Deployment](#deployment)

<!-- markdown-toc end -->


## 1. Workspace Architecture

### Go Workspace Configuration

**File:** `go.work`

```
use (
    .
    ./contracts
    ./contracts/definitions/foomod
    ./contracts/definitions/barmod
    ./modules/foomod
    ./modules/barmod
    ./libs/ogl
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
mmw/
├── go.work                               # Workspace coordinator
├── cmd/
│   └── mmw/
│       └── main.go                       # Composition root: wires all modules
├── contracts/                            # go module: mmw-contracts
│   ├── go.mod
│   ├── buf.yaml
│   ├── buf.gen.yaml
│   ├── definitions/
│   │   ├── foomod/                       # go module: ZERO dependencies
│   │   │   ├── go.mod
│   │   │   ├── api.go                    # FooService interface
│   │   │   ├── dto.go                    # Request/response types
│   │   │   ├── errors.go                 # Public error sentinels
│   │   │   └── inproc_client.go          # Wraps any FooService impl
│   │   └── barmod/
│   ├── proto/
│   │   └── foo/v1/foo.proto
│   └── gen/go/foo/v1/
├── modules/
│   ├── foomod/                           # go module: mmw-foomod
│   │   ├── go.mod
│   │   ├── foomod.go                     # Module factory + Infrastructure struct
│   │   └── internal/
│   └── barmod/
├── libs/
│   └── ogl/                              # go module: ogl (mmw-platform)
│       └── platform/
│           ├── core/app.go               # Module interface
│           ├── runner.go                 # platform.App + Run()
│           ├── server/
│           ├── events/
│           └── connect/
└── test/e2e/
    └── go.mod
```

### Module Dependency Rules

**Enforced by Go compiler:**

1. **Contract Definitions:** Zero dependencies
   ```go
   // contracts/definitions/foomod/go.mod
   module github.com/example/mmw-contracts/definitions/foomod
   go 1.23
   // NO require statements
   ```

2. **Modules:** Can import contract definitions only (not other modules' internals)
   ```go
   // modules/barmod/go.mod
   module github.com/example/mmw-barmod
   require (
       github.com/example/mmw-contracts/definitions/foomod v0.0.0
   )
   // Cannot import github.com/example/mmw-foomod/internal (compiler error)
   ```

3. **Internal Packages:** Cannot be imported across module boundaries
   - `modules/foomod/internal/` is inaccessible to `barmod`
   - Enforced by Go's internal package rules + separate modules

## 2. Contract Definition Layer

### Contract Structure

**Location:** `contracts/definitions/foomod/`

**Purpose:** Public API contract that **defines** in-process communication with module independence.

### Contract Components

**1. Interface Definition** (`api.go`)

```go
package deffoomod

import "context"

// FooService defines the public API contract for foomod.
type FooService interface {
    CreateFoo(ctx context.Context, req CreateFooRequest) (*FooDTO, error)
    GetFoo(ctx context.Context, id string) (*FooDTO, error)
    ListFoos(ctx context.Context, ownerID string) ([]*FooDTO, error)
}
```

**2. Data Transfer Objects** (`dto.go`)

```go
package deffoomod

type FooDTO struct {
    ID       string
    Title    string
    Status   string
    Priority string
    OwnerID  string
}

type CreateFooRequest struct {
    Title    string
    Priority string
    OwnerID  string
}
```

**3. Error Types** (`errors.go`)

```go
package deffoomod

import "errors"

var (
    ErrFooNotFound    = errors.New("foo not found")
    ErrFooUnavailable = errors.New("foo service unavailable")
)
```

**4. In-Process Client** (`inproc_client.go`)

```go
package deffoomod

import "context"

// InprocClient wraps any FooService implementation for in-process calls.
// In the composition root this is the *foomod.Module concrete type,
// but the client holds the interface — not the concrete type.
type InprocClient struct {
    impl FooService
}

func NewInprocClient(impl FooService) *InprocClient {
    return &InprocClient{impl: impl}
}

func (c *InprocClient) CreateFoo(ctx context.Context, req CreateFooRequest) (*FooDTO, error) {
    return c.impl.CreateFoo(ctx, req)
}

func (c *InprocClient) GetFoo(ctx context.Context, id string) (*FooDTO, error) {
    return c.impl.GetFoo(ctx, id)
}

func (c *InprocClient) ListFoos(ctx context.Context, ownerID string) ([]*FooDTO, error) {
    return c.impl.ListFoos(ctx, ownerID)
}
```

### Contract Dependency Rules

**What contracts contain:**
- Interface definitions
- DTOs (pure data structures, no business logic)
- Error constants
- InprocClient (thin wrapper that delegates to the interface)

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

## 3. Module Structure

### Directory Layout

```
modules/foomod/
├── go.mod                            # Independent module: mmw-foomod
├── foomod.go                         # Module factory + Infrastructure struct
├── cmd/
│   └── foomod/
│       └── main.go                   # Standalone entry (optional)
└── internal/                         # Private implementation
    ├── domain/                       # Domain layer (aggregate, value objects, events)
    ├── application/                  # Application layer (commands, queries, ports)
    │   ├── command/
    │   ├── query/
    │   ├── ports/
    │   └── dto/
    ├── adapters/                     # Adapters layer (I/O boundaries)
    │   ├── inbound/
    │   │   └── connect/              # Connect RPC handler
    │   └── outbound/
    │       ├── persistence/postgres/ # Database adapter (pgxpool, no ORM)
    │       └── events/               # Outbox event dispatcher
    └── infra/
        ├── config/
        └── persistence/migrations/
```

### go.mod Configuration

```go
module github.com/example/mmw-foomod

go 1.23

require (
    // Contract this module implements
    github.com/example/mmw-contracts/definitions/foomod v0.0.0

    // Platform library
    github.com/ovya/ogl v0.0.0

    // DB driver
    github.com/jackc/pgx/v5 v5.7.0

    // Connect RPC
    connectrpc.com/connect v1.18.1

    // Generated proto types
    github.com/example/mmw-contracts v0.0.0
)
```

### Internal Package Organization

**Domain Layer:** `internal/domain/`
- Pure business logic, zero external dependencies
- Aggregate root with private fields, public methods, domain event emission
- Value objects with validation in constructors
- `ToSnapshot()` / `FromSnapshot()` for persistence without exposing private state
- `AllEvents` slice for event type registration

**Application Layer:** `internal/application/`
- `command/` — write operations (use UoW + repo + dispatcher)
- `query/` — read operations (no UoW needed)
- `ports/` — secondary port interfaces (`FooRepository`, `EventDispatcher`, `UnitOfWork`)
- `dto/` — application-level data transfer objects

**Adapters Layer:** `internal/adapters/`
- `inbound/connect/` — Connect RPC handler implementing generated interface
- `outbound/persistence/postgres/` — repository using `*pgxpool.Pool` + `ogluow.GetExecutor`
- `outbound/events/` — outbox dispatcher writing to `foo.event` table

**Infrastructure Layer:** `internal/infra/`
- `config/` — module-specific config loading
- `persistence/migrations/` — SQL migration files

## 4. Hexagonal Architecture Layers

### Layer Dependencies

```
Domain (pure business logic)
    ↑
Application (use cases, ports)
    ↑
Adapters (implementations)
    ↑
foomod.go (Infrastructure struct + Module factory)
    ↑
cmd/mmw/main.go (composition root)
```

**Rule:** Dependencies point inward. Inner layers never depend on outer layers.

### Domain Layer

**Location:** `modules/foomod/internal/domain/`

**Aggregate Root** — enforces all business invariants:
```go
// modules/foomod/internal/domain/foo.go
type Foo struct {
    id        FooID
    title     FooTitle
    status    FooStatus
    priority  Priority
    ownerID   uuid.UUID
    createdAt time.Time
    updatedAt time.Time
    events    []DomainEvent
}

func NewFoo(title FooTitle, priority Priority, ownerID uuid.UUID) (*Foo, error) {
    if ownerID == uuid.Nil {
        return nil, ErrOwnerRequired
    }
    now := time.Now().UTC()
    f := &Foo{
        id: NewFooID(), title: title,
        status: StatusPending, priority: priority,
        ownerID: ownerID, createdAt: now, updatedAt: now,
    }
    f.events = append(f.events, FooCreated{FooID: f.id, OwnerID: ownerID, At: now})
    return f, nil
}

func (f *Foo) Complete() error {
    if f.status == StatusCompleted { return ErrAlreadyCompleted }
    if f.status == StatusCancelled { return ErrCancelledCannotComplete }
    f.status = StatusCompleted
    f.updatedAt = time.Now().UTC()
    f.events = append(f.events, FooCompleted{FooID: f.id, At: f.updatedAt})
    return nil
}

func (f *Foo) PopEvents() []DomainEvent { evts := f.events; f.events = nil; return evts }
```

**Value Objects** — validated named types:
```go
type FooID struct{ v uuid.UUID }
func NewFooID() FooID              { return FooID{v: uuid.New()} }
func (id FooID) String() string    { return id.v.String() }

type FooTitle struct{ v string }
func NewFooTitle(s string) (FooTitle, error) {
    if len(s) == 0   { return FooTitle{}, ErrTitleEmpty }
    if len(s) > 200  { return FooTitle{}, ErrTitleTooLong }
    return FooTitle{v: s}, nil
}
```

**Properties:**
- No external dependencies (standard library only)
- Fully testable without infrastructure
- State changes only through methods that validate and emit events

### Application Layer

**Location:** `modules/foomod/internal/application/`

**Secondary Ports** (`ports/`):
```go
// modules/foomod/internal/application/ports/ports.go
type FooRepository interface {
    Save(ctx context.Context, foo *domain.Foo) error
    FindByID(ctx context.Context, id domain.FooID) (*domain.Foo, error)
    Delete(ctx context.Context, id domain.FooID) error
    Health(ctx context.Context) error
}

type EventDispatcher interface {
    Dispatch(ctx context.Context, events []domain.DomainEvent) error
}

type UnitOfWork interface {
    Do(ctx context.Context, fn func(ctx context.Context) error) error
}
```

**Application Service** (thin facade):
```go
// modules/foomod/internal/application/foo_service.go
type FooApplicationService struct {
    repo       ports.FooRepository
    uow        ports.UnitOfWork
    dispatcher ports.EventDispatcher
}

func NewFooApplicationService(
    repo ports.FooRepository,
    uow ports.UnitOfWork,
    dispatcher ports.EventDispatcher,
) *FooApplicationService {
    return &FooApplicationService{repo: repo, uow: uow, dispatcher: dispatcher}
}
```

**Command Handler** (write operation with UoW):
```go
// modules/foomod/internal/application/command/create_foo.go
func (h *CreateFooHandler) Handle(ctx context.Context, cmd CreateFooCommand) (*FooDTO, error) {
    var result *FooDTO
    err := h.uow.Do(ctx, func(ctx context.Context) error {
        title, err := domain.NewFooTitle(cmd.Title)
        if err != nil { return err }
        priority := domain.Priority(cmd.Priority)
        ownerID, err := uuid.Parse(cmd.OwnerID)
        if err != nil { return domain.ErrInvalidFooID }

        foo, err := domain.NewFoo(title, priority, ownerID)
        if err != nil { return err }
        if err := h.repo.Save(ctx, foo); err != nil { return err }
        if err := h.dispatcher.Dispatch(ctx, foo.PopEvents()); err != nil { return err }

        result = FooDTOFromDomain(foo)
        return nil
    })
    return result, err
}
```

**Properties:**
- Orchestrates domain objects and ports
- No knowledge of HTTP, database drivers, or frameworks
- UoW ensures transactional safety for write operations
- Query handlers are read-only and skip UoW

### Adapters Layer

**Location:** `modules/foomod/internal/adapters/`

#### Inbound Adapters

**Connect RPC Handler** (`adapters/inbound/connect/foo_handler.go`):

```go
package connect

import (
    "connectrpc.com/connect"
    foov1 "github.com/example/mmw-contracts/gen/go/foo/v1"
    "github.com/example/mmw-contracts/gen/go/foo/v1/foov1connect"
)

// FooHandler implements the generated foov1connect.FooServiceHandler interface.
type FooHandler struct {
    svc *application.FooApplicationService
}

// Compile-time assertion
var _ foov1connect.FooServiceHandler = (*FooHandler)(nil)

func (h *FooHandler) CreateFoo(
    ctx context.Context,
    req *connect.Request[foov1.CreateFooRequest],
) (*connect.Response[foov1.CreateFooResponse], error) {
    dto, err := h.svc.CreateFoo(ctx, application.CreateFooCommand{
        Title:    req.Msg.Title,
        Priority: req.Msg.Priority,
        OwnerID:  ownerIDFromContext(ctx),
    })
    if err != nil {
        return nil, domainErrToConnect(err)
    }
    return connect.NewResponse(&foov1.CreateFooResponse{
        Foo: fooToProto(dto),
    }), nil
}
```

**Error mapping:**

| Domain error | Connect status |
|---|---|
| `domain.ErrNotFound` | `connect.CodeNotFound` |
| `domain.ErrTitleEmpty` | `connect.CodeInvalidArgument` |
| `domain.ErrAlreadyCompleted` | `connect.CodeFailedPrecondition` |
| other | `connect.CodeInternal` |

#### Outbound Adapters

**Postgres Repository** (`adapters/outbound/persistence/postgres/`):

```go
// PostgresFooRepository implements ports.FooRepository.
type PostgresFooRepository struct {
    pool *pgxpool.Pool
}

var _ ports.FooRepository = (*PostgresFooRepository)(nil)

func (r *PostgresFooRepository) Save(ctx context.Context, foo *domain.Foo) error {
    snap := foo.ToSnapshot()
    exec := ogluow.GetExecutor(ctx, r.pool) // uses tx if one is active
    _, err := exec.Exec(ctx,
        `INSERT INTO foo.foo (id, title, status, priority, owner_id, created_at, updated_at)
         VALUES ($1, $2, $3, $4, $5, $6, $7)`,
        snap.ID, snap.Title, snap.Status, snap.Priority, snap.OwnerID,
        snap.CreatedAt, snap.UpdatedAt,
    )
    return err
}
```

Key points:
- No ORM — direct `pgx` queries
- `ogluow.GetExecutor(ctx, pool)` returns the active transaction executor or falls back to pool
- `ToSnapshot()` / `FromSnapshot()` bridge between domain private fields and the DB

**Outbox Dispatcher** (`adapters/outbound/events/`):

```go
// PostgresOutboxDispatcher writes events to foo.event in the same transaction.
func (d *PostgresOutboxDispatcher) Dispatch(ctx context.Context, events []domain.DomainEvent) error {
    exec := ogluow.GetExecutor(ctx, d.pool)
    for _, evt := range events {
        payload, _ := json.Marshal(evt)
        _, err := exec.Exec(ctx,
            `INSERT INTO foo.event (aggregate_id, event_type, payload) VALUES ($1, $2, $3)`,
            aggregateID(evt), evt.EventType(), payload,
        )
        if err != nil { return err }
    }
    return nil
}
```

Events are written **in the same transaction** as the business data. A background `EventsRelay` polls the outbox table and publishes to Watermill. This prevents silent event loss on crash between DB commit and publish. See [cross-service-events.md](cross-service-events.md).

### Infrastructure Layer

**Location:** `modules/foomod/internal/infra/`

**Configuration:** `infra/config/config.go`

Each module loads its own configuration:

```go
cfg, err := config.Load(ctx, "")
```

**Environment Variables:**

```bash
# Required (secrets - never in files)
DB_PASSWORD=secret_password

# Optional (runtime overrides)
APP_ENV=prod              # Selects prod.yaml
HTTP_PORT=:8080
DB_HOST=db.prod.internal
```

## 5. Runtime Orchestration

### mmw-platform: Module Lifecycle

**mmw-platform** (`libs/ogl/platform`) provides the `core.Module` interface and the `platform.App` runner. Every feature module must implement this interface:

```go
// libs/ogl/platform/core/app.go
type Module interface {
    Start(ctx context.Context) error
    Close() error
}
```

Each module exposes:

- `Infrastructure` struct — lists ALL shared resources the module needs (DB pool, event bus, logger, other module contracts)
- `Module` struct — holds internal components (HTTP server, outbox relay, logger)
- `New(Infrastructure) (*Module, error)` — wires all internals, returns ready-to-run module
- `Start(ctx) error` — starts HTTP server + outbox relay via internal errgroup
- `Close() error` — releases any module-specific resources
- `var _ oglcore.Module = (*Module)(nil)` — compile-time interface assertion

See [mmw-platform.md](mmw-platform.md) for the full reference.

### Composition Root (main.go)

**Location:** `cmd/mmw/main.go`

```go
func main() {
    ctx, cancel := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
    defer cancel()

    // 1. Shared resources
    pool, _ := pgxpool.New(ctx, os.Getenv("DATABASE_URL"))
    defer pool.Close()
    logger := slog.Default()
    rawBus := gochannel.NewGoChannel(gochannel.Config{}, watermill.NewSlogLogger(logger))
    eventBus := oglevents.NewWatermillBus(rawBus)

    // 2. Module factories (dependency order matters)
    fooModule, _ := foomod.New(foomod.Infrastructure{
        DBPool:   pool,
        EventBus: eventBus,
        Logger:   logger,
    })

    barModule, _ := barmod.New(barmod.Infrastructure{
        DBPool:   pool,
        EventBus: eventBus,
        FooSvc:   deffoomod.NewInprocClient(fooModule),
        Logger:   logger,
    })

    // 3. Register event topics each module publishes
    for _, topic := range foomod.NotifyEvents {
        eventBus.RegisterTopic(topic)
    }

    // 4. Run all modules (shared-fate errgroup)
    app := platform.New(fooModule, barModule)
    if err := app.Run(ctx); err != nil {
        logger.Error("app shutdown", "err", err)
        os.Exit(1)
    }
}
```

**Wiring flow:**
1. Shared resources (DB pool, event bus, logger) created once
2. Module factories called in dependency order (providers before consumers)
3. `InprocClient` wraps the `*Module` — the contract interface, not the concrete type
4. `platform.App.Run(ctx)` starts all modules under a shared-fate errgroup

### Supervision with errgroup

**Pattern:** Shared fate architecture — if one module fails, all shut down together.

Concrete example when foomod fails:

- **WITHOUT shared fate**
  - foomod: Crashed (database connection fails)
  - barmod: Running (but calls to foomod fail silently)
  - Result: Zombie monolith in undefined state

- **WITH shared fate (platform.App)**
  - foomod `Start()` returns error
  - platform errgroup cancels context for all modules
  - barmod `Start()` detects `ctx.Done()` → graceful shutdown
  - Process exits
  - Kubernetes restarts entire pod with clean state

`platform.App.Run()` internally uses `errgroup.WithContext` across all module `Start()` calls. Each module's internal errgroup (HTTP server + relay) propagates errors up through `Start()` to the platform runner.

**Alternative:** For fine-grained fault tolerance, use [github.com/thejerf/suture](https://github.com/thejerf/suture) for Erlang-style supervision trees.

## 6. Protobuf Integration

### When to Use Protobuf

Use protobuf when modules need network transport. Skip for in-process only.

### Directory Structure

```
contracts/
├── proto/                  # Source of truth (clean)
│   ├── buf.yaml
│   └── foo/v1/
│       └── foo.proto
└── gen/                    # Generated code (do not edit)
    └── go/foo/v1/
        ├── foo.pb.go
        └── foov1connect/foo.connect.go
```

### Protobuf Definition

**File:** `contracts/proto/foo/v1/foo.proto`

```protobuf
syntax = "proto3";
package foo.v1;
option go_package = "github.com/example/mmw-contracts/gen/go/foo/v1;foov1";

message Foo {
  string id = 1;
  string title = 2;
  string status = 3;
  string priority = 4;
  string owner_id = 5;
}

service FooService {
  rpc CreateFoo(CreateFooRequest) returns (CreateFooResponse);
  rpc GetFoo(GetFooRequest) returns (GetFooResponse);
}
```

### Code Generation

**File:** `contracts/buf.gen.yaml`

```yaml
version: v2
plugins:
  - remote: buf.build/protocolbuffers/go
    out: gen/go
    opt: paths=source_relative
  - remote: buf.build/connectrpc/go
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

The generated `foov1connect` package provides the `FooServiceHandler` interface that the Connect inbound adapter must implement:

```go
// modules/foomod/internal/adapters/inbound/connect/foo_handler.go

// Compile-time assertion — fails build if FooHandler is missing any method
var _ foov1connect.FooServiceHandler = (*FooHandler)(nil)
```

**Transport swapping:** The same `FooHandler` works for both network (Connect over HTTP) and in-process (InprocClient) transport. The composition root decides which path to use — no application code changes. See [protobuf-contracts.md](protobuf-contracts.md).

## 7. Testing & Operations

### Test Organization

**Domain unit tests:** Co-located with domain code
- Location: `modules/foomod/internal/domain/*_test.go`
- Purpose: Business rules, value object validation, state transitions
- Dependencies: None (pure functions, no mocks needed)

**Application unit tests:** Co-located with handlers
- Location: `modules/foomod/internal/application/**/*_test.go`
- Purpose: Command/query handlers with mock ports
- Dependencies: Mock implementations of `FooRepository`, `EventDispatcher`, `UnitOfWork`

**Adapter integration tests:** Co-located with adapters
- Location: `modules/foomod/internal/adapters/**/*_test.go`
- Purpose: Repository + real DB (testcontainers), Connect handler
- Dependencies: Real database via testcontainers

**E2E tests:** Root-level cross-module tests
- Location: `test/e2e/`
- Purpose: Full HTTP request → DB → response journeys
- Dependencies: Full system

### Operational Commands

```bash
# Run full workspace tests
go test ./...

# Build the monolith binary
go build -o bin/mmw ./cmd/mmw

# Run standalone module
go run ./modules/foomod/cmd/foomod

# Verify module dependencies
go mod verify

# Update all dependencies
go get -u ./...

# Tidy all dependencies
go mod tidy

# Generate protobuf code
cd contracts && buf generate

# Lint proto files
cd contracts && buf lint
```

### Architecture Validation

**Tool:** `arch-test` (in `tools/arch-test/`)

See [arch-test implementation](./arch-test.md)

```bash
# Validate architectural boundaries
go run ./tools/arch-test

# Checks:
# - Contract definitions have zero dependencies
# - No module imports another module's internal/
# - Dependency flow rules (domain ← application ← adapters)
# - libs/ogl never imports modules/
```

## Deployment

**Monolith Mode (default):**
```bash
# Build single binary running all modules in one process
go build -o bin/mmw ./cmd/mmw
```

**Distributed Mode:**

To distribute modules, swap the `InprocClient` for a network Connect client in `main.go`. Only the wiring changes; application code stays unchanged.

**Before (In-Process):**
```go
// cmd/mmw/main.go
fooModule, _ := foomod.New(foomod.Infrastructure{...})
barModule, _ := barmod.New(barmod.Infrastructure{
    FooSvc: deffoomod.NewInprocClient(fooModule), // in-process
})
```

**After (Network/Connect):**
```go
// barmod deployed separately; foomod exposed over HTTP
barModule, _ := barmod.New(barmod.Infrastructure{
    FooSvc: deffoomod.NewConnectClient("https://foomod.internal"), // network
})
```

The `FooApplicationService` inside barmod calls the same `FooService` interface — it never knows whether the transport is in-process or network.
