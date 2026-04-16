# Agent Context: mmw Architecture Reference

This document defines the architectural rules for AI agents generating
or modifying code in this repository. All code generation must comply
with these constraints.

## Repository Layout

```
poc/
├── go.work                            ← Go workspace coordinating all modules
├── cmd/mmw/main.go                    ← ONLY place for composition root wiring
├── contracts/
│   ├── proto/                         ← Proto definitions (source of truth)
│   │   ├── todo/v1/todo.proto
│   │   ├── auth/v1/auth.proto
│   │   └── options/v1/options.proto   ← custom topic/error annotations
│   ├── go/
│   │   ├── network/                   ← generated: proto structs + Connect stubs
│   │   │   ├── todo/v1/todo.pb.go
│   │   │   └── todo/v1/todov1connect/
│   │   └── application/               ← generated: application-layer contracts
│   │       ├── todo/                  ← TodoService interface, events, errors
│   │       └── auth/                  ← AuthPrivateService, AuthPublicService, connect_client.go
│   ├── buf.gen.todo.yaml
│   ├── buf.gen.auth.yaml
│   └── cmd/protoc-gen-go-contracts/   ← custom buf plugin source
├── modules/
│   ├── todo/                          ← github.com/pivaldi/mmw-todo
│   ├── auth/                          ← github.com/pivaldi/mmw-auth
│   └── notifications/                 ← github.com/pivaldi/mmw-notifications
├── mmw/                               ← github.com/piprim/mmw (platform + CLI)
│   ├── pkg/platform/                  ← shared platform library
│   └── cmd/mmw-cli/                   ← mmw check arch, mmw new module, etc.
└── libs/ogl/                          ← ogl shared utilities
```

## Module Import Paths

| Module | Go module path |
|---|---|
| Platform library | `github.com/piprim/mmw` |
| Contracts (network + application) | `github.com/pivaldi/mmw-contracts` |
| Todo module | `github.com/pivaldi/mmw-todo` |
| Auth module | `github.com/pivaldi/mmw-auth` |
| Notifications module | `github.com/pivaldi/mmw-notifications` |

## Module Structure (non-negotiable)

Every feature module in `modules/<name>/` must have:

1. **`<name>.go` at the module root** — contains:
   - `type Infrastructure struct` — lists ALL shared dependencies as interfaces
   - `type Module struct` — holds server, relay, router, logger (all private)
   - `var _ pfcore.Module = (*Module)(nil)` — compile-time assertion
   - `const ModuleName = "..."` and `const PGSchema = "..."`
   - `func New(infra Infrastructure) (*Module, error)` — wires all internals
   - `func (m *Module) Start(ctx context.Context) error` — runs HTTP server + outbox relay + event router
   - `func Migrate(ctx context.Context, pool *pgxpool.Pool) error` — for tests and migration tooling

2. **`internal/domain/`** — zero external dependencies; no imports from adapters, application, or infra
3. **`internal/application/`** — depends only on domain and port interfaces; NO database or HTTP imports
   - `service.go` — `TodoService` interface (primary port) + `TodoApplicationService` implementation
   - `ports/repository.go` — `TodoRepository` interface (secondary port)
   - `ports/events.go` — `EventDispatcher` interface (secondary port)
   - `command/` — command handlers (writes: use `ports.UnitOfWork.WithTransaction`)
   - `query/` — query handlers (reads: no transactions, no events)
   - `dto/` — application-layer data transfer types (NOT domain types, NOT proto types)
4. **`internal/adapters/inbound/connect/`** — Connect RPC handler + `NewTokenValidator` (auth middleware)
5. **`internal/adapters/inbound/inproc/`** — `ContractAdapter` implementing the generated contract interface
6. **`internal/adapters/outbound/persistence/postgres/`** — repository (`*pfuow.UnitOfWork`, pgx only, no ORM)
7. **`internal/adapters/outbound/events/`** — `PostgresOutboxDispatcher` + `domainTopics` map
8. **`internal/infra/`** — config loading + DB migrations (embedded FS)

## Layer Dependency Rules

```
domain          ← no imports from any other internal layer
application     ← imports domain only (+ ports it defines)
adapters        ← imports application (via ports/interfaces) and domain
infra           ← imports only stdlib and config/migration libs
<module>.go     ← imports all internal layers (wiring only)
```

**NEVER:**
- `domain` importing anything from `application` or `adapters`
- `application` importing from `adapters` or infrastructure packages (`pgx`, `net/http`, `connectrpc`)
- `adapters` importing from `infra`
- Any module importing another module's `internal/` packages
- A contract package importing a feature module (structurally impossible: would be circular)

## Contract Generation Rules

Contracts are **generated automatically** — never hand-written application interfaces.

```bash
cd poc/contracts

# 1. Build the custom plugin (once):
go build -o $(go env GOPATH)/bin/protoc-gen-go-contracts ./cmd/protoc-gen-go-contracts

# 2. Standard plugins (network types + Connect stubs → go/network/):
buf generate --template buf.gen.todo.yaml
buf generate --template buf.gen.auth.yaml

# 3. Custom plugin (application-layer contracts → go/application/):
buf generate --template buf.gen.yaml
```

Generated files per domain (`contracts/go/application/<domain>/`):

| File | Content |
|---|---|
| `*_contract_gen.go` | Service interface + `NoopXxxService` stub + sentinel error |
| `events_gen.go` | `TopicXxx` constants + `var Topics []string` + type aliases |
| `errors_gen.go` | `ErrorCodeXxx` constants (aliases over proto enum values) |
| `connect_client.go` | `NewPrivateHTTPClient` / `NewPublicHTTPClient` (hand-written, not regenerated) |

## Platform Runner

See [mmw-platform.md](mmw-platform.md) for the complete reference.

**Interface (from `mmw/pkg/platform/core/app.go`):**

```go
import pfcore "github.com/piprim/mmw/pkg/platform/core"

type Module interface {
    Start(ctx context.Context) error
}
```

**Required pattern in `cmd/mmw/main.go`:**

```go
import "github.com/piprim/mmw/pkg/platform"

modules := []pfcore.Module{authModule, todoModule, notifModule}
platform.New(logger, modules).Run(ctx)
```

**Module dependency order:** instantiate the depended-on module first.
Inject cross-module dependencies via the module's accessor method:

```go
// Auth has no inter-module deps — instantiate first
authModule, _ := auth.New(auth.Infrastructure{
    DBPool: dbPool, EventBus: eventBus, Logger: logger.With("module", auth.ModuleName),
})

// Todo depends on auth — wire via PrivateService() accessor
todoModule, _ := todo.New(todo.Infrastructure{
    DBPool:     dbPool,
    EventBus:   eventBus,
    Subscriber: rawBus,
    Logger:     logger.With("module", todo.ModuleName),
    AuthSvc:    authModule.PrivateService(), // returns defauth.AuthPrivateService (interface)
})
```

The accessor returns a `ContractAdapter` that translates between proto-typed
contract requests and the internal DTO-based application service. The consumer
sees only the generated contract interface — never the concrete module type.

## In-Process Contract Adapter

Each module exposes a `ContractAdapter` in `internal/adapters/inbound/inproc/`
that wraps the internal `application.XxxService` and implements the generated
`tododef.TodoService` (or `authdef.AuthPrivateService`) interface:

```go
// modules/todo/internal/adapters/inbound/inproc/contract_adapter.go
type ContractAdapter struct{ svc application.TodoService }

var _ tododef.TodoService = (*ContractAdapter)(nil)

func NewContractAdapter(svc application.TodoService) *ContractAdapter {
    return &ContractAdapter{svc: svc}
}
```

The module exposes it via an accessor on `*Module`:

```go
// modules/auth/auth.go
func (m *Module) PrivateService() authdef.AuthPrivateService {
    return inproc.NewContractAdapter(m.service)
}
```

## Event Pattern Rules

1. Domain events are **written to the outbox table** inside the Unit of Work transaction — never published directly to Watermill
2. `pfoutbox.NewEnventsRelay` is instantiated in `New()` and started in `Module.Start()` to relay outbox → Watermill
3. The `domainTopics` map (in `adapters/outbound/events/topics.go`) is the **single place** where domain event type strings are mapped to Watermill routing keys (topic constants from `events_gen.go`)
4. Subscriber modules (notifications) receive the raw `*gochannel.GoChannel`, not the wrapped `SystemEventBus`
5. Topic constants used for subscriptions come from `contracts/go/application/<domain>/events_gen.go`; pass `append(tododef.Topics, authdef.Topics...)` to the notifications module

```go
// Relay instantiation (inside New()):
relay := pfoutbox.NewEnventsRelay(infra.DBPool, infra.EventBus, infra.Logger, "todo.event")

// domainTopics (adapters/outbound/events/topics.go):
var domainTopics = map[string]string{
    domain.EventTypeCreated:   tododef.TopicUserTaskCreated,
    domain.EventTypeCompleted: tododef.TopicUserTaskCompleted,
    // …
}
```

## Repository Pattern

Repositories accept a `*pfuow.UnitOfWork`, not a raw `*pgxpool.Pool`.
`uow.Executor(ctx)` returns the active `pgx.Tx` (inside `WithTransaction`)
or a pool connection otherwise:

```go
import (
    pfdb  "github.com/piprim/mmw/pkg/platform/db"
    pfuow "github.com/piprim/mmw/pkg/platform/pg/uow"
)

// Save uses named args derived from the snapshot's db-tagged fields.
func (r *PostgresTodoRepository) Save(ctx context.Context, todo *domain.Todo) error {
    _, err := r.uow.Executor(ctx).Exec(ctx, query, pgx.NamedArgs(pfdb.StructArgs(todo.Snapshot())))
    return eris.Wrap(err, "saving todo")
}
```

`pfdb.StructArgs(snap)` reflects `db`-tagged struct fields into `map[string]any`
compatible with `pgx.NamedArgs`. Use `pgx.CollectOneRow(rows, pgx.RowToStructByName[domain.XxxSnapshot])`
for reads.

## Naming Conventions

| Thing | Convention |
|---|---|
| Module package | module name (e.g. `todo`, `auth`) |
| Module constant | `const ModuleName = "Todo"` |
| Outbox table | `<schema>.event` (e.g. `todo.event`, `auth.event`) |
| Contract import alias | `defauth`, `deftodo` (prefix `def`) |
| Contract accessor | `authModule.PrivateService()` returns `defauth.AuthPrivateService` |
| Platform import alias | `pfcore`, `pfuow`, `pfoutbox`, `pfevents`, `pfserver`, `pfconnect` |

## Forbidden Patterns

- `var db *sql.DB` — use `*pgxpool.Pool`
- ORM usage (GORM, ent) — use raw `pgx` queries with `pgx.NamedArgs`
- Direct Watermill publish in command handlers — use outbox dispatcher
- `http.ListenAndServe` — use `pfserver.NewHTTPServer`
- Importing `modules/<x>/internal/` from any other module
- Hand-writing contract interfaces — generate them from proto via `protoc-gen-go-contracts`
- `defXxx.NewInprocClient(module)` — use `module.PrivateService()` or `module.PublicService()` instead
- Storing the proto-typed request directly in the domain or application layer — map to `dto.XxxRequest` in the adapter
