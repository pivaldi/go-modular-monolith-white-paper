# Complete Directory Structure

Full annotated layout of an `mmw`-style monorepo with two feature
modules (`foomod`, `barmod`), a shared library (`ogl`), and a contract
module.

```
mmw/
├── go.work                               ← workspace: coordinates all modules
├── go.work.sum
│
├── config/                               ← shared app config (root module)
│   └── config.go                         ← mmwconfig.Load() — DB URL, log level, etc.
│
├── cmd/
│   └── mmw/
│       └── main.go                       ← composition root: wires all modules
│
├── contracts/                            ← go module: mmw-contracts
│   ├── go.mod                            ← module github.com/example/mmw-contracts
│   ├── buf.yaml                          ← buf lint + breaking change config
│   ├── buf.gen.yaml                      ← connect-go + protobuf code gen config
│   ├── definitions/
│   │   ├── foomod/                       ← go module: mmw-contracts/definitions/foomod
│   │   │   ├── go.mod                    ← ZERO dependencies
│   │   │   ├── api.go                    ← FooService interface
│   │   │   ├── dto.go                    ← request/response types
│   │   │   ├── errors.go                 ← public error sentinels
│   │   │   └── inproc_client.go          ← wraps any FooService impl behind interface
│   │   └── barmod/                       ← go module: mmw-contracts/definitions/barmod
│   │       ├── go.mod
│   │       ├── api.go
│   │       ├── dto.go
│   │       ├── errors.go
│   │       └── inproc_client.go
│   ├── proto/
│   │   ├── foo/v1/foo.proto
│   │   └── bar/v1/bar.proto
│   └── gen/
│       └── go/
│           ├── foo/v1/
│           │   ├── foo.pb.go
│           │   └── foov1connect/foo.connect.go
│           └── bar/v1/
│               ├── bar.pb.go
│               └── barv1connect/bar.connect.go
│
├── modules/
│   ├── foomod/                           ← go module: mmw-foomod
│   │   ├── go.mod
│   │   ├── foomod.go                     ← Module{}, Infrastructure{}, New(), Start()
│   │   ├── cmd/foomod/main.go            ← optional standalone entry point
│   │   └── internal/
│   │       ├── domain/
│   │       │   ├── foo.go                ← Foo aggregate root
│   │       │   ├── value_objects.go      ← FooID, FooTitle, FooStatus, Priority
│   │       │   ├── events.go             ← domain events + AllEvents slice
│   │       │   ├── snapshot.go           ← FooSnapshot, ToSnapshot, FromSnapshot
│   │       │   └── errors.go             ← domain error sentinels
│   │       ├── application/
│   │       │   ├── service.go            ← FooApplicationService (facade)
│   │       │   ├── command/
│   │       │   │   ├── create_foo.go     ← CreateFooHandler
│   │       │   │   ├── complete_foo.go   ← CompleteFooHandler
│   │       │   │   └── delete_foo.go     ← DeleteFooHandler
│   │       │   ├── query/
│   │       │   │   ├── get_foo.go        ← GetFooHandler
│   │       │   │   └── list_foos.go      ← ListFoosHandler
│   │       │   ├── dto/
│   │       │   │   └── foo_dto.go        ← commands, queries, FooDTO
│   │       │   └── ports/
│   │       │       ├── repository.go     ← FooRepository interface
│   │       │       ├── events.go         ← EventDispatcher interface
│   │       │       └── uow.go            ← UnitOfWork interface
│   │       ├── adapters/
│   │       │   ├── inbound/
│   │       │   │   └── connect/
│   │       │   │       ├── foo_handler.go       ← Connect RPC handler
│   │       │   │       ├── auth_middleware.go   ← JWT validation middleware
│   │       │   │       └── errors.go            ← domain error → Connect code
│   │       │   └── outbound/
│   │       │       ├── persistence/
│   │       │       │   └── postgres/
│   │       │       │       └── foo_repository.go ← PostgresFooRepository
│   │       │       └── events/
│   │       │           └── outbox_dispatcher.go  ← PostgresOutboxDispatcher
│   │       └── infra/
│   │           ├── config/
│   │           │   └── config.go         ← module-level config (port, env)
│   │           └── persistence/
│   │               └── migrations/
│   │                   ├── 001_create_foo_table.sql
│   │                   └── 002_create_event_table.sql
│   │
│   └── barmod/                           ← go module: mmw-barmod (same layout)
│       ├── go.mod
│       ├── barmod.go
│       └── internal/ ...
│
├── libs/
│   └── ogl/                              ← go module: ogl (shared library)
│       ├── go.mod
│       ├── platform/
│       │   ├── core/
│       │   │   └── app.go                ← core.Module interface
│       │   ├── runner.go                 ← platform.App + Run()
│       │   ├── server/
│       │   │   └── server.go             ← oglserver.HTTPServer (health, debug routes)
│       │   ├── events/
│       │   │   ├── bus.go                ← SystemEventBus interface
│       │   │   └── watermill.go          ← WatermillBus implementation
│       │   ├── connect/
│       │   │   └── interceptors.go       ← error logging interceptor
│       │   └── middleware/
│       │       ├── cors.go
│       │       ├── logging.go
│       │       └── recovery.go
│       ├── pg/
│       │   └── uow/                      ← Unit of Work (transaction management)
│       ├── db/
│       │   └── outbox/                   ← EventsRelay (outbox poller/publisher)
│       ├── slog/                         ← structured logger setup
│       └── config/                       ← config loading helpers
│
├── deployments/
│   └── docker-compose.yml                ← PostgreSQL + any other infra
│
├── tools/
│   └── arch-test/
│       └── main.go                       ← import boundary validator
│
└── test/
    └── e2e/
        ├── go.mod
        └── foo_test.go                   ← end-to-end tests (HTTP → DB)
```

## Test Organisation

| Layer | Location | What it tests |
|---|---|---|
| Domain unit | `modules/foomod/internal/domain/*_test.go` | Business rules, value object validation, state transitions |
| Application unit | `modules/foomod/internal/application/**/*_test.go` | Command/query handlers with mock ports |
| Adapter integration | `modules/foomod/internal/adapters/**/*_test.go` | Repository + DB (testcontainers), Connect handler |
| Contract | `contracts/definitions/foomod/*_test.go` | InprocClient satisfies interface |
| E2E | `test/e2e/` | Full HTTP request → DB → response |
