# mmw-platform: The Module Runner

`mmw-platform` is the `mmw/pkg/platform` package (module path
`github.com/piprim/mmw`). It provides the runtime that boots every module
in the monolith, runs them concurrently, and shuts them all down cleanly
when any one fails or the process receives a signal.

## 1. Overview

A monolith built with this architecture is a single binary containing
multiple isolated modules. Each module owns its HTTP server, database
connection usage, and background workers. The platform runner holds
them together with a shared context and an errgroup.

```
Signal (SIGTERM/SIGINT)
       │
       ▼
  context.Cancel()
       │
       ├── Module: auth.Start(ctx)          ──► HTTP server + outbox relay
       ├── Module: todo.Start(ctx)          ──► HTTP server + outbox relay + event router
       └── Module: notifications.Start(ctx) ──► event subscriber
```

**Shared fate:** if any module returns an error from `Start`, the
platform cancels the shared context, causing all other modules to stop.

## 2. `core.Module` Interface

Every module must satisfy this single interface:

```go
// mmw/pkg/platform/core/app.go
package core

import "context"

// Module defines the contract for all isolated applications in the monolith.
type Module interface {
    // Start runs the module until ctx is canceled or a fatal error occurs.
    // It must block. Returning nil means clean shutdown. Returning an error
    // causes the entire platform to stop.
    Start(ctx context.Context) error
}
```

There are no `Stop()`, `Close()`, or `Shutdown()` methods. Shutdown is
signaled entirely via context cancellation.

## 3. `platform.App` and `Run()`

```go
// mmw/pkg/platform/runner.go
package platform

import (
    "context"
    "log/slog"

    "github.com/piprim/mmw/pkg/platform/core"
    "github.com/rotisserie/eris"
    "golang.org/x/sync/errgroup"
)

type App struct {
    logger  *slog.Logger
    modules []core.Module
}

func New(logger *slog.Logger, modules []core.Module) *App {
    return &App{logger: logger, modules: modules}
}

func (a *App) Run(ctx context.Context) error {
    a.logger.Info("starting platform runner")
    g, gCtx := errgroup.WithContext(ctx)

    for _, m := range a.modules {
        g.Go(func() error {
            return eris.Wrapf(m.Start(gCtx), "module failed")
        })
    }

    if err := g.Wait(); err != nil {
        a.logger.Error("application stopped with error", "err", err)
        return eris.Wrap(err, "application stopped with error")
    }

    a.logger.Info("application stopped gracefully")
    return nil
}
```

`Run` blocks until all modules exit. The errgroup context (`gCtx`) is
passed to every module. If `auth` fails, `gCtx` is canceled, which
propagates to `todo` and `notifications`, triggering their shutdown.

## 4. `Infrastructure` Struct Pattern

Each module defines its own `Infrastructure` struct listing the shared
resources it needs. This is the only public API surface of the module
besides `New` and `Start`.

```go
// modules/todo/todo.go
type Infrastructure struct {
    DBPool     *pgxpool.Pool
    EventBus   pfevents.SystemEventBus    // Watermill bus wrapper (write side)
    Subscriber message.Subscriber        // raw GoChannel (read side)
    AuthSvc    defauth.AuthPrivateService // cross-module contract (injected)
    Logger     *slog.Logger
}
```

**Rules:**
- Only accept what the module actually uses — no "just in case" fields
- Cross-module dependencies come in as contract interfaces (`defauth.AuthPrivateService`), never as concrete `*Module` pointers
- The DB pool is shared across modules; each module manages its own schema

## 5. Module Factory: `New`

`New` is the composition root of a single module. It wires all internal
components and returns a ready-to-run `*Module`.

```go
// modules/todo/todo.go
func New(infra Infrastructure) (*Module, error) {
    cfg, err := config.Load(context.Background(), "")
    if err != nil {
        return nil, eris.Wrap(err, "app failed to load config")
    }

    // 1. Application service (repo + UoW + event dispatcher)
    todoService := newApplicationService(infra)

    // 2. Inbound event router (Watermill)
    router, err := newEventRouter(infra)
    if err != nil { return nil, err }

    // 3. HTTP server with Connect RPC handler + auth middleware
    httpServer := newHTTPServer(cfg, infra, todoService)

    return &Module{
        relay:   pfoutbox.NewEnventsRelay(infra.DBPool, infra.EventBus, infra.Logger, "todo.event"),
        server:  httpServer,
        router:  router,
        logger:  infra.Logger,
        service: todoService,
    }, nil
}

func newApplicationService(infra Infrastructure) application.TodoService {
    uow := pfuow.New(infra.DBPool)
    todoRepo := postgres.NewPostgresTodoRepository(uow)
    eventDispatcher := events.NewPostgresOutboxDispatcher(uow)
    return application.NewTodoApplicationService(todoRepo, uow, eventDispatcher)
}
```

Everything inside `New` is private wiring. Nothing leaks out except
the `*Module` handle.

## 6. `Start()` Implementation

Each module runs its own internal `errgroup` to manage its server and
background workers:

```go
// modules/todo/todo.go
type Module struct {
    relay   *pfoutbox.EventsRelay
    server  *pfserver.HTTPServer
    router  *message.Router
    logger  *slog.Logger
    service application.TodoService
}

// Compile-time assertion: Module must satisfy core.Module
var _ pfcore.Module = (*Module)(nil)

func (m *Module) Start(ctx context.Context) error {
    m.logger.Info("starting the app")
    g, gCtx := errgroup.WithContext(ctx)

    g.Go(func() error { return m.server.Start(gCtx) })

    if m.relay != nil {
        g.Go(func() error { m.relay.Start(gCtx); return nil })
    }

    g.Go(func() error { return m.router.Run(gCtx) })

    return eris.Wrapf(g.Wait(), "%s failure", ModuleName)
}

func (m *Module) Close() error {
    m.logger.Info("shutting down module internal resources")
    return nil
}
```

## 7. Composition Root Walkthrough

The composition root in `cmd/mmw/main.go`:

1. Sets up the signal context (SIGTERM/SIGINT → cancel)
2. Initializes config + structured logger
3. Opens the shared DB pool
4. Creates the Watermill raw bus (`gochannel.GoChannel`) and wraps it as `SystemEventBus`
5. Instantiates modules **in dependency order** — the depended-on module first
6. Wires cross-module contracts via the module's accessor method (e.g. `authModule.PrivateService()`)
7. Hands all modules to `platform.New().Run(ctx)`

```go
// cmd/mmw/main.go
func initModules(
    logger *slog.Logger,
    dbPool *pgxpool.Pool,
    rawBus *gochannel.GoChannel,
    eventBus pfevents.SystemEventBus,
) ([]pfcore.Module, error) {
    // 1. Auth — no inter-module dependencies
    authModule, err := auth.New(auth.Infrastructure{
        DBPool:   dbPool,
        EventBus: eventBus,
        Logger:   logger.With("module", auth.ModuleName),
    })
    if err != nil { return nil, err }

    // 2. Todo — depends on auth's private service for JWT validation
    todoModule, err := todo.New(todo.Infrastructure{
        DBPool:     dbPool,
        EventBus:   eventBus,
        Subscriber: rawBus,
        Logger:     logger.With("module", todo.ModuleName),
        AuthSvc:    authModule.PrivateService(), // returns defauth.AuthPrivateService
    })
    if err != nil { return nil, err }

    // 3. Notifications — subscribes to all domain events
    notifModule, err := notifications.New(notifications.Infrastructure{
        Subscriber:  rawBus,
        Logger:      logger.With("module", notifications.ModuleName),
        Topics:      append(tododef.Topics, authdef.Topics...),
        WithNotifer: true,
    })
    if err != nil { return nil, err }

    return []pfcore.Module{todoModule, authModule, notifModule}, nil
}
```

## 8. Transport Swapping (In-Process → Network)

To split a module out as a separate process, replace the in-process accessor
with an HTTP client in the composition root. No application code changes:

**Before (monolith — in-process):**
```go
todoModule, _ := todo.New(todo.Infrastructure{
    AuthSvc: authModule.PrivateService(), // direct method call
})
```

**After (distributed — Connect over HTTP):**
```go
// auth runs in its own pod at https://auth.internal
todoModule, _ := todo.New(todo.Infrastructure{
    AuthSvc: defauth.NewPrivateHTTPClient(
        authv1connect.NewAuthPrivateServiceClient(&http.Client{}, "https://auth.internal"),
    ),
})
```

The `defauth.AuthPrivateService` interface is the same in both cases.
The todo module never knows which transport is used.

## 9. Compile-Time Safety

Each module file includes this assertion immediately after the struct definition:

```go
var _ pfcore.Module = (*Module)(nil)
```

If `Start(ctx context.Context) error` is missing or has the wrong
signature, the build fails. This is the first line of architecture
enforcement — no runtime surprises.

## 10. Adding a New Module — Checklist

- [ ] Create `modules/newmod/newmod.go` with `Infrastructure`, `Module`, `New`, `Start`, and the compile-time assertion (`var _ pfcore.Module = (*Module)(nil)`)
- [ ] Write `.proto` service definition and generate contracts via `buf generate --template buf.gen.newmod.yaml`
- [ ] Add `modules/newmod` to `go.work` under the `poc/` workspace
- [ ] Add `modules/newmod` to `cmd/mmw/main.go`: instantiate in dependency order, inject contract interfaces, append to `modules` slice
- [ ] Add `newmod`'s `Topics` slice to the notifications module subscription list if it emits events
- [ ] Add DB migrations under `modules/newmod/internal/infra/persistence/migrations/`
- [ ] Verify `go build ./cmd/mmw/...` compiles cleanly
