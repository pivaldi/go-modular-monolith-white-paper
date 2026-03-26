# Composition Root: Module Pattern with mmw-platform Supervisor

The monolith's `cmd/mmw/main.go` performs direct, explicit wiring of all
modules and hands them to `platform.New().Run(ctx)`. Each module encapsulates
its own HTTP server, outbox relay, and background workers behind a single
`Start(ctx) error` method.

See [mmw-platform.md](mmw-platform.md) for the full platform runner reference.

## The Module Struct

Each module in `modules/foosvc/foosvc.go` follows this structure:

```go
package foosvc

import (
    "context"
    "log/slog"

    "golang.org/x/sync/errgroup"
    ogloutbox "github.com/ovya/ogl/db/outbox"
    oglcore "github.com/ovya/ogl/platform/core"
    oglevents "github.com/ovya/ogl/platform/events"
    oglserver "github.com/ovya/ogl/platform/server"
    "github.com/jackc/pgx/v5/pgxpool"
    defbar "github.com/example/mmw-contracts/definitions/barsvc"
    "github.com/example/mmw-foosvc/internal/domain"
)

const ModuleName = "foosvc"

var NotifyEvents = domain.AllEvents

// Infrastructure holds all shared resources injected from the composition root.
type Infrastructure struct {
    DBPool   *pgxpool.Pool
    EventBus oglevents.SystemEventBus
    BarSvc   defbar.BarService  // cross-module dependency via contract
    Logger   *slog.Logger
}

// Module is the runtime handle returned by New. It satisfies core.Module.
type Module struct {
    relay  *ogloutbox.EventsRelay
    server *oglserver.HTTPServer
    logger *slog.Logger
}

// Compile-time assertion: *Module must satisfy the core.Module interface.
var _ oglcore.Module = (*Module)(nil)
```

## The Factory Function

`New` wires all internal components. Nothing from inside `modules/foosvc/internal/`
is accessible outside the module — the `internal/` package rule enforces this.

```go
func New(infra Infrastructure) (*Module, error) {
    cfg, err := config.Load(context.Background(), "")
    if err != nil {
        return nil, fmt.Errorf("foosvc: load config: %w", err)
    }

    // 1. Outbound adapters
    repo := postgres.NewFooRepository(infra.DBPool)
    dispatcher := events.NewPostgresOutboxDispatcher(infra.DBPool)
    uow := ogluow.New(infra.DBPool)

    // 2. Application service
    appSvc := application.NewFooApplicationService(repo, uow, dispatcher)

    // 3. Inbound adapter (Connect RPC handler)
    mux := http.NewServeMux()
    path, handler := foov1connect.NewFooServiceHandler(
        connectadapter.NewFooHandler(appSvc),
        connect.WithInterceptors(oglconnect.NewErrorLoggingInterceptor(infra.Logger)),
    )
    mux.Handle(path, connectadapter.NewAuthMiddleware(
        infra.BarSvc, infra.Logger, nil, handler,
    ))

    // 4. HTTP mux + auth middleware
    // 5. HTTP server
    srv := oglserver.NewHTTPServer(oglserver.HTTPServerInfra{
        Config:    cfg.Server,
        Handler:   mux,
        Logger:    infra.Logger,
        HealthFns: oglserver.HealthFns{"database": appSvc.Health},
    })

    // 6. Outbox relay (polls foo.event table, publishes to EventBus)
    relay := ogloutbox.NewEnventsRelay(infra.DBPool, infra.EventBus, infra.Logger, "foo.event")

    return &Module{
        relay:  relay,
        server: srv,
        logger: infra.Logger,
    }, nil
}
```

## The Start Method

`Start` blocks until the context is canceled or a fatal error occurs.
It uses its own internal `errgroup` to run the HTTP server and outbox
relay concurrently:

```go
func (m *Module) Start(ctx context.Context) error {
    m.logger.Info("starting foosvc")
    g, gCtx := errgroup.WithContext(ctx)

    g.Go(func() error {
        return m.server.Start(gCtx)
    })

    // relay is nil when the module has no outbox table (e.g. event-only modules or test doubles)
    if m.relay != nil {
        g.Go(func() error {
            m.relay.Start(gCtx)
            return nil
        })
    }

    return g.Wait()
}
```

## The Composition Root

`cmd/mmw/main.go` creates shared resources once, instantiates modules
in dependency order, then hands them all to the platform runner:

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
    "github.com/ovya/ogl/platform"
    oglcore "github.com/ovya/ogl/platform/core"
    oglevents "github.com/ovya/ogl/platform/events"
    oglslog "github.com/ovya/ogl/slog"

    barsvc "github.com/example/mmw-barsvc"
    defbar "github.com/example/mmw-contracts/definitions/barsvc"
    foosvc "github.com/example/mmw-foosvc"
    notifications "github.com/example/mmw-notifications"
    mmwconfig "github.com/example/mmw/config"
)

func main() {
    ctx, cancel := signal.NotifyContext(context.Background(),
        os.Interrupt, syscall.SIGTERM)
    defer cancel()

    // 1. Shared resources
    cfg, _ := mmwconfig.Load(ctx)
    logger, _ := oglslog.New(oglslog.HandlerText, cfg.LogLevel.SlogLevel())
    dbPool, _ := pgxpool.New(ctx, cfg.MainDatabase.URL())
    defer dbPool.Close()

    rawBus := gochannel.NewGoChannel(
        gochannel.Config{OutputChannelBuffer: 1024, Persistent: true},
        watermill.NewSlogLogger(logger),
    )
    defer rawBus.Close()
    systemBus := oglevents.NewWatermillBus(rawBus)

    // 2. Instantiate modules in dependency order (depended-on first)
    barModule, _ := barsvc.New(barsvc.Infrastructure{
        DBPool:   dbPool,
        EventBus: systemBus,
        Logger:   logger.With("module", barsvc.ModuleName),
    })

    fooModule, _ := foosvc.New(foosvc.Infrastructure{
        DBPool:   dbPool,
        EventBus: systemBus,
        Logger:   logger.With("module", foosvc.ModuleName),
        BarSvc:   defbar.NewInprocClient(barModule), // in-process wiring
    })

    notifEvents := append(foosvc.NotifyEvents, barsvc.NotifyEvents...)
    notifModule, _ := notifications.New(notifications.Infrastructure{
        Subscriber: rawBus,
        Logger:     logger.With("module", notifications.ModuleName),
        Topics:     notifEvents,
    })

    // 3. Start all modules under the platform runner
    modules := []oglcore.Module{barModule, fooModule, notifModule}
    if err := platform.New(logger, modules).Run(ctx); err != nil {
        logger.Error("platform error", "err", err)
        os.Exit(1)
    }
}
```

## Shared-Fate Shutdown

The `errgroup` inside `platform.Run` shares a single context (`gCtx`).
When any module's `Start` returns an error, `gCtx` is canceled. Every
other module receives this cancellation and exits cleanly via its own
context-aware server and relay.

This means:
- A database crash in `foosvc` stops the entire process — not just `foosvc`
- A clean `SIGTERM` reaches all modules simultaneously
- No module can silently fail and keep others running in a degraded state

## Error Handling in main

Production composition roots wrap every `New` call:

```go
barModule, err := barsvc.New(barsvc.Infrastructure{ ... })
if err != nil {
    logger.Error("failed to initialize barsvc", "err", err)
    os.Exit(1)
}
```

If a module fails to initialize, the process exits before the platform
runner starts — no partial startup.
