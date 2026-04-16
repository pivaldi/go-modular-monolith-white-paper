# Composition Root: Module Pattern with mmw-platform Supervisor

The monolith's `cmd/mmw/main.go` performs direct, explicit wiring of all
modules and hands them to `platform.New().Run(ctx)`. Each module encapsulates
its own HTTP server, outbox relay, and background workers behind a single
`Start(ctx) error` method.

See [mmw-platform.md](mmw-platform.md) for the full platform runner reference.

## The Module Struct

Each module (e.g. `modules/todo/todo.go`) follows this structure:

```go
package todo

import (
    "context"
    "log/slog"

    "golang.org/x/sync/errgroup"
    pfcore    "github.com/piprim/mmw/pkg/platform/core"
    pfoutbox  "github.com/piprim/mmw/pkg/platform/db/outbox"
    pfevents  "github.com/piprim/mmw/pkg/platform/events"
    pfserver  "github.com/piprim/mmw/pkg/platform/server"
    "github.com/jackc/pgx/v5/pgxpool"
    "github.com/ThreeDotsLabs/watermill/message"
    defauth "github.com/pivaldi/mmw-contracts/go/application/auth"
)

const ModuleName = "Todo"

// Infrastructure holds all shared resources injected from the composition root.
type Infrastructure struct {
    DBPool     *pgxpool.Pool
    EventBus   pfevents.SystemEventBus  // write side (for outbox relay)
    Subscriber message.Subscriber       // read side (for event router)
    AuthSvc    defauth.AuthPrivateService  // cross-module dependency via contract
    Logger     *slog.Logger
}

// Module is the runtime handle returned by New. It satisfies core.Module.
type Module struct {
    relay   *pfoutbox.EventsRelay
    server  *pfserver.HTTPServer
    router  *message.Router
    logger  *slog.Logger
    service application.TodoService
}

// Compile-time assertion: *Module must satisfy the core.Module interface.
var _ pfcore.Module = (*Module)(nil)
```

## The Factory Function

`New` wires all internal components. Nothing from inside `modules/todo/internal/`
is accessible outside the module — the `internal/` package rule enforces this.

```go
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
        // Outbox relay: polls todo.event every 2 s → publishes to SystemEventBus
        relay:   pfoutbox.NewEnventsRelay(infra.DBPool, infra.EventBus, infra.Logger, "todo.event"),
        server:  httpServer,
        router:  router,
        logger:  infra.Logger,
        service: todoService,
    }, nil
}
```

## The Start Method

`Start` blocks until the context is canceled or a fatal error occurs.
It uses its own internal `errgroup` to run the HTTP server, outbox relay,
and Watermill event router concurrently:

```go
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

## The Composition Root

`cmd/mmw/main.go` creates shared resources once, instantiates modules in dependency
order, then hands them all to the platform runner:

```go
package main

import (
    "context"
    "os"
    "os/signal"
    "syscall"

    "github.com/ThreeDotsLabs/watermill"
    "github.com/ThreeDotsLabs/watermill/pubsub/gochannel"
    "github.com/jackc/pgx/v5/pgxpool"
    "github.com/piprim/mmw/pkg/platform"
    pfcore   "github.com/piprim/mmw/pkg/platform/core"
    pfevents "github.com/piprim/mmw/pkg/platform/events"
    pfslog   "github.com/piprim/mmw/pkg/platform/slog"

    auth          "github.com/pivaldi/mmw-auth"
    notifications "github.com/pivaldi/mmw-notifications"
    todo          "github.com/pivaldi/mmw-todo"
    authdef       "github.com/pivaldi/mmw-contracts/go/application/auth"
    tododef       "github.com/pivaldi/mmw-contracts/go/application/todo"
    mmwconfig     "github.com/pivaldi/mmw/config"
)

var exitCode = 0

func main() {
    ctx, cancel := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
    var dbPool *pgxpool.Pool

    defer func() {
        if dbPool != nil { dbPool.Close() }
        cancel()
        os.Exit(exitCode)
    }()

    // 1. Config + logger
    config, logger, err := initObservability(ctx)
    if err != nil { return }

    // 2. Shared infrastructure
    dbPool, err = getDatabasePoolConnexion(ctx, logger, config.MainDatabase.URL())
    if err != nil { exitCode = 1; return }

    rawBus := gochannel.NewGoChannel(
        gochannel.Config{OutputChannelBuffer: 1024, Persistent: true},
        watermill.NewSlogLogger(logger),
    )
    defer rawBus.Close()
    eventBus := pfevents.NewWatermillBus(rawBus)

    // 3. Modules in dependency order
    modules, err := initModules(logger, dbPool, rawBus, eventBus)
    if err != nil { return }

    // 4. Start all modules under the platform runner (shared-fate errgroup)
    logger.Info("Platform startup…")
    if err = platform.New(logger, modules).Run(ctx); err != nil {
        exitCode = 1
        logger.Error("platform error", "err", err)
    }
}

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
        AuthSvc:    authModule.PrivateService(), // returns defauth.AuthPrivateService (interface!)
    })
    if err != nil { return nil, err }

    // 3. Notifications — subscribes to all domain events from auth and todo
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

## Shared-Fate Shutdown

The `errgroup` inside `platform.Run` shares a single context (`gCtx`).
When any module's `Start` returns an error, `gCtx` is canceled. Every
other module receives this cancellation and exits cleanly.

This means:
- A database crash in `auth` stops the entire process — not just `auth`
- A clean `SIGTERM` reaches all modules simultaneously
- No module can silently fail and keep others running in a degraded state

## Transport Swapping (In-Process → Network)

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

The `defauth.AuthPrivateService` interface is the same in both cases. The todo module
never knows which transport is used.
