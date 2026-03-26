# Agent Context: mmw Architecture Reference

This document defines the architectural rules for AI agents generating
or modifying code in this repository. All code generation must comply
with these constraints.

## Repository Layout

```
mmw/
├── go.work
├── cmd/mmw/main.go            ← ONLY place for composition root wiring
├── contracts/definitions/     ← ZERO-dependency contract modules
├── modules/                   ← Feature modules (one Go module each)
├── libs/ogl/                  ← Shared platform library
└── deployments/               ← Infrastructure (Docker, k8s)
```

## Module Structure (non-negotiable)

Every feature module in `modules/<name>/` must have:

1. **`<name>.go` at the module root** — contains:
   - `type Infrastructure struct` — lists ALL shared dependencies
   - `type Module struct` — holds server, relay, logger (private)
   - `var _ oglcore.Module = (*Module)(nil)` — compile-time assertion
   - `func New(infra Infrastructure) (*Module, error)` — wires all internals
   - `func (m *Module) Start(ctx context.Context) error` — runs HTTP server + relay

2. **`internal/domain/`** — zero external dependencies; no imports from adapters, application, or infra
3. **`internal/application/`** — depends only on domain and port interfaces; NO database or HTTP imports
4. **`internal/adapters/inbound/connect/`** — Connect RPC handler + auth middleware
5. **`internal/adapters/outbound/persistence/postgres/`** — repository (pgx only, no ORM)
6. **`internal/adapters/outbound/events/`** — `PostgresOutboxDispatcher`
7. **`internal/infra/`** — config loading + DB migrations

## Layer Dependency Rules

```
domain          ← no imports from other internal layers
application     ← imports domain only
adapters        ← imports application (via ports) and domain
infra           ← imports application ports for config types
<module>.go     ← imports all internal layers (wiring only)
```

**NEVER:**
- `domain` importing anything from `application` or `adapters`
- `application` importing from `adapters` or infrastructure packages (`pgx`, `net/http`)
- `adapters` importing from `infra`
- Any module importing another module's `internal/` packages

## Contract Definition Rules

Every `contracts/definitions/<svc>/go.mod` must have **zero `require` entries**.

Required files: `api.go`, `dto.go`, `errors.go`, `inproc_client.go`.

The `InprocClient` must forward all interface methods to `impl` with no
added logic.

## Platform Runner (mmw-platform)

See [mmw-platform.md](mmw-platform.md) for the complete reference.

**Required pattern in `cmd/mmw/main.go`:**

```go
modules := []oglcore.Module{barModule, fooModule, notifModule}
platform.New(logger, modules).Run(ctx)
```

**Module dependency order:** instantiate the module being depended on
first. Inject cross-module dependencies via `NewInprocClient`:

```go
barModule, _ := barsvc.New(...)                          // no cross-module deps
fooModule, _ := foosvc.New(foosvc.Infrastructure{        // depends on barsvc
    BarSvc: defbarsvc.NewInprocClient(barModule),
    ...
})
```

## Event Pattern Rules

1. Domain events are **written to the outbox table** inside the Unit of Work transaction — never published directly to Watermill
2. `EventsRelay` (from `ogl/db/outbox`) is started in `Module.Start` to relay outbox → Watermill
3. Each module exports `var NotifyEvents = domain.AllEvents` for the composition root to collect topics
4. The notifications module (or any subscriber) receives the raw `*gochannel.GoChannel`, not the wrapped bus

## Naming Conventions

| Thing | Convention |
|---|---|
| Module package | `foosvc` (lowercase, no separator) |
| Module constant | `const ModuleName = "foosvc"` |
| Outbox table | `<schema>.event` (e.g. `foo.event`) |
| Contract package | `deffoosvc` (import alias) |
| InprocClient call | `deffoosvc.NewInprocClient(fooModule)` |

## Forbidden Patterns

- `var db *sql.DB` — use `*pgxpool.Pool`
- ORM usage (GORM, ent) — use raw `pgx` queries
- Direct Watermill publish in command handlers — use outbox
- `http.ListenAndServe` — use `oglserver.NewHTTPServer`
- Importing `modules/<x>/internal/` from any other module
