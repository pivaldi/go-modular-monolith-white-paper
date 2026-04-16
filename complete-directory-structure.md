# Complete Directory Structure

Full annotated layout of the `poc/` monorepo with three feature modules
(`auth`, `todo`, `notifications`), a shared utility library (`ogl`), a
platform library + CLI (`mmw`), and the contracts module.

```
poc/
├── go.work                               ← workspace: coordinates all modules
├── go.work.sum
├── go.mod                                ← root module: github.com/pivaldi/mmw
│
├── config/
│   └── configs/                          ← root TOML configs (default.toml, development.toml, …)
│
├── cmd/
│   └── mmw/
│       └── main.go                       ← composition root: wires all modules
│
├── contracts/                            ← go module: github.com/pivaldi/mmw-contracts
│   ├── go.mod                            ← only depends on connect + protobuf
│   ├── buf.yaml                          ← buf lint + breaking change config
│   ├── buf.gen.yaml                      ← standard connect-go + protobuf code gen
│   ├── buf.gen.auth.yaml                 ← auth-specific generation (incl. contracts plugin)
│   ├── buf.gen.todo.yaml                 ← todo-specific generation (incl. contracts plugin)
│   ├── cmd/
│   │   └── protoc-gen-go-contracts/      ← custom protoc plugin
│   │       └── main.go                   ← generates interfaces, events, errors from proto
│   ├── proto/                            ← source of truth (edit these)
│   │   ├── auth/v1/auth.proto            ← auth service definitions + events + error codes
│   │   ├── todo/v1/todo.proto            ← todo service definitions + events + error codes
│   │   ├── common/v1/common.proto        ← DomainError, shared message types
│   │   └── options/v1/options.proto      ← (options.v1.topic) proto extension
│   ├── go/                               ← generated Go code (do not edit)
│   │   ├── application/
│   │   │   ├── auth/                     ← generated application contracts for auth
│   │   │   │   ├── auth_private_service_contract_gen.go  ← AuthPrivateService interface + noop
│   │   │   │   ├── auth_public_service_contract_gen.go   ← AuthPublicService interface + noop
│   │   │   │   ├── connect_client.go     ← PublicHTTPClient + PrivateHTTPClient
│   │   │   │   ├── errors_gen.go         ← error code constants (if proto has error enum)
│   │   │   │   ├── events_gen.go         ← TopicXxx constants + Topics slice + type aliases
│   │   │   │   └── types.go              ← hand-written supplemental types (if any)
│   │   │   └── todo/                     ← generated application contracts for todo
│   │   │       ├── todo_service_contract_gen.go
│   │   │       ├── errors_gen.go
│   │   │       └── events_gen.go
│   │   └── network/                      ← standard protobuf + connect generated code
│   │       ├── auth/v1/
│   │       │   ├── auth.pb.go            ← protobuf message structs
│   │       │   └── authv1connect/
│   │       │       └── auth.connect.go   ← AuthPublicServiceHandler + AuthPrivateServiceHandler interfaces + clients
│   │       └── todo/v1/
│   │           ├── todo.pb.go
│   │           └── todov1connect/
│   │               └── todo.connect.go   ← TodoServiceHandler interface + client
│   └── ts/                               ← generated TypeScript types (for frontend)
│       ├── auth/
│       └── todo/
│
├── libs/
│   └── ogl/                              ← go module: ogl (utility library)
│       ├── go.mod
│       ├── file/                         ← file utilities (zip, lock)
│       ├── os/                           ← OS helpers
│       └── string/                       ← string helpers
│
├── mmw/                                  ← go module: github.com/piprim/mmw (platform + CLI)
│   ├── go.mod
│   ├── pkg/
│   │   ├── platform/                     ← runtime platform library
│   │   │   ├── core/
│   │   │   │   └── app.go                ← core.Module interface
│   │   │   ├── runner.go                 ← platform.App + Run()
│   │   │   ├── safego.go                 ← safe goroutine launcher
│   │   │   ├── errors.go                 ← DomainError, platform error codes
│   │   │   ├── root_repo.go              ← workspace root discovery
│   │   │   ├── server/
│   │   │   │   └── server.go             ← HTTPServer (health, pprof, gRPC reflection)
│   │   │   ├── events/
│   │   │   │   ├── bus.go                ← SystemEventBus interface
│   │   │   │   └── watermill.go          ← WatermillBus implementation
│   │   │   ├── connect/
│   │   │   │   └── interceptors.go       ← NewErrorLoggingInterceptor
│   │   │   ├── middleware/
│   │   │   │   ├── middleware.go         ← Middleware type alias
│   │   │   │   ├── logging.go            ← LoggingMiddleware (request-id, status-based level)
│   │   │   │   ├── recovery.go           ← RecoveryMiddleware (panic → 500)
│   │   │   │   ├── cors.go               ← CORSMiddleware (Connect-compatible)
│   │   │   │   └── auth.go               ← BearerAuthMiddleware (TokenValidator)
│   │   │   ├── authctx/
│   │   │   │   └── authctx.go            ← WithUserID / UserID context helpers
│   │   │   ├── config/
│   │   │   │   ├── config.go             ← NewContext, Fill
│   │   │   │   ├── base.go               ← Base struct (GetAppEnv)
│   │   │   │   ├── database.go           ← Database struct + URL()
│   │   │   │   ├── port.go               ← Port type
│   │   │   │   ├── environment.go        ← Environment enum (dev/staging/prod/testing)
│   │   │   │   └── doc.go
│   │   │   ├── db/
│   │   │   │   ├── struct_args.go        ← StructArgs (db-tagged fields → map[string]any)
│   │   │   │   ├── migrator/             ← Goose wrapper
│   │   │   │   └── outbox/
│   │   │   │       └── relay.go          ← EventsRelay (polls outbox, publishes to bus)
│   │   │   ├── pg/
│   │   │   │   └── uow/
│   │   │   │       └── uow.go            ← UnitOfWork (pgxpool + pgx.Tx via WithTransaction)
│   │   │   ├── infra/
│   │   │   │   └── ieventbus/
│   │   │   │       └── watermill.go
│   │   │   ├── os/
│   │   │   │   └── os.go
│   │   │   └── slog/
│   │   │       └── slog.go               ← New(HandlerText|HandlerJSON, level)
│   │   ├── archtest/                     ← architectural boundary validator
│   │   │   ├── archtest.go               ← RunAll entry point + Validator interface
│   │   │   ├── custom/                   ← custom validators (contract purity, lib deps, etc.)
│   │   │   ├── orchestrator/             ← discovers modules, runs mise arch:check per module
│   │   │   └── reporter/                 ← formats and prints results
│   │   └── scaffold/                     ← cookiecutter-style module generator
│   │       ├── generator.go              ← GenerateModule / GenerateContract
│   │       ├── manifest.go               ← LoadManifest (reads template.toml)
│   │       ├── options.go                ← Variable, NormalizeKey, EnrichVars
│   │       ├── workspace.go              ← go.work and mise.toml updaters
│   │       └── _templates/              ← embedded Go templates
│   │           ├── template.toml         ← variables + conditions manifest
│   │           └── modules/{{.Name}}/
│   │               ├── go.mod.tmpl
│   │               ├── mise.toml
│   │               └── {{.Name}}mod.go
│   └── cmd/
│       └── mmw-cli/                      ← mmw CLI binary
│           ├── main.go
│           └── cmd/
│               ├── root.go               ← mmw (root command)
│               ├── new/
│               │   ├── new.go            ← mmw new
│               │   ├── module.go         ← mmw new module [--template]
│               │   └── contract.go       ← mmw new contract <name>
│               ├── check/
│               │   ├── check.go          ← mmw check
│               │   └── arch.go           ← mmw check arch
│               └── test/
│                   ├── root.go           ← mmw test
│                   └── coverage.go       ← mmw test coverage
│
├── modules/
│   ├── auth/                             ← go module: github.com/pivaldi/mmw-auth
│   │   ├── go.mod
│   │   ├── auth.go                       ← Module{}, Infrastructure{}, New(), Start()
│   │   ├── cmd/migration/config.go       ← standalone migration entry point
│   │   └── internal/
│   │       ├── domain/
│   │       │   ├── user.go               ← User aggregate root
│   │       │   ├── session.go            ← Session aggregate root
│   │       │   ├── objects.go            ← value objects (Email, Password, UserID, …)
│   │       │   ├── events.go             ← domain events (UserRegistered, UserDeleted, …)
│   │       │   ├── snapshot.go           ← ToSnapshot / FromSnapshot
│   │       │   └── errors.go             ← domain error sentinels
│   │       ├── application/
│   │       │   ├── service.go            ← AuthApplicationService
│   │       │   └── ports/
│   │       │       └── repository.go     ← UserRepository, SessionRepository interfaces
│   │       ├── adapters/
│   │       │   ├── inbound/
│   │       │   │   ├── connect/          ← Connect RPC handler (AuthPublicService + AuthPrivateService)
│   │       │   │   └── inproc/           ← InprocClient adapter (wraps AuthApplicationService)
│   │       │   └── outbound/
│   │       │       ├── persistence/postgres/   ← UserRepository + SessionRepository (pgx)
│   │       │       └── events/           ← PostgresOutboxDispatcher
│   │       └── infra/
│   │           ├── config/               ← module config loader (TOML + env)
│   │           └── persistence/migrations/  ← embedded Goose SQL migrations
│   │
│   ├── todo/                             ← go module: github.com/pivaldi/mmw-todo
│   │   ├── go.mod
│   │   ├── todo.go                       ← Module{}, Infrastructure{}, New(), Start()
│   │   ├── cmd/migration/config.go
│   │   └── internal/
│   │       ├── domain/
│   │       │   ├── todo.go               ← Todo aggregate root
│   │       │   ├── value_objects.go      ← Title, Description, Status, DueDate
│   │       │   ├── value_objects_enum.go ← Status enum
│   │       │   ├── events.go             ← domain events (TodoCreated, TodoCompleted, …)
│   │       │   ├── snapshot.go           ← TodoSnapshot, ToSnapshot / FromSnapshot
│   │       │   └── errors.go             ← domain error sentinels
│   │       ├── application/
│   │       │   ├── service.go            ← TodoService interface + TodoApplicationService
│   │       │   ├── command/              ← write handlers (CreateTodo, UpdateTodo, DeleteTodo, …)
│   │       │   ├── query/                ← read handlers (GetTodo, ListTodos)
│   │       │   ├── dto/                  ← application DTOs
│   │       │   ├── ports/
│   │       │   │   ├── repository.go     ← TodoRepository interface
│   │       │   │   ├── events.go         ← EventDispatcher interface
│   │       │   │   └── mocks/            ← mockery-generated mocks (ports only)
│   │       │   └── authctx/              ← application-layer auth context helpers
│   │       ├── adapters/
│   │       │   ├── inbound/
│   │       │   │   ├── connect/          ← Connect RPC handler + TokenValidator
│   │       │   │   └── events/           ← Watermill inbound handlers (HandleUserDeleted)
│   │       │   └── outbound/
│   │       │       ├── persistence/postgres/ ← PostgresTodoRepository (pgx + StructArgs)
│   │       │       └── events/           ← PostgresOutboxDispatcher
│   │       ├── infra/
│   │       │   ├── config/
│   │       │   └── persistence/migrations/
│   │       └── ...
│   │           └── test/contract/        ← contract tests (provider-side)
│   │
│   └── notifications/                    ← go module: github.com/pivaldi/mmw-notifications
│       ├── go.mod
│       ├── notifications.go              ← Module{}, Infrastructure{}, New(), Start()
│       └── internal/
│           └── infra/persistence/migrations/
│
├── deployments/
│   └── README.md
│
└── docs/
    ├── plans/                            ← implementation plans
    ├── presentations/                    ← architecture slides + HTML presentation
    └── superpowers/                      ← specs + plans from AI-assisted sessions
```

## Test Organisation

| Layer | Location | What it tests |
|---|---|---|
| Domain unit | `modules/todo/internal/domain/*_test.go` | Business rules, value object validation, state transitions |
| Application unit | `modules/todo/internal/application/**/*_test.go` | Command/query handlers with mocked ports |
| Adapter integration | `modules/todo/internal/adapters/**/*_test.go` | Repository + real DB (testcontainers), Connect handler |
| Contract | `modules/todo/test/contract/contract_test.go` | Module satisfies its own contract interface |
| System | `poc/test/system/` | Full HTTP → DB → event → response flows |
