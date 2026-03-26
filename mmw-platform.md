# mmw-platform: The Module Runner

`mmw-platform` is the `ogl/platform` package. It provides the runtime
that boots every module in the monolith, runs them concurrently, and
shuts them all down cleanly when any one fails or the process receives
a signal.

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
       ├── Module: foomod.Start(ctx) ──► HTTP server + outbox relay
       ├── Module: barmod.Start(ctx) ──► HTTP server + outbox relay
       └── Module: baznotif.Start(ctx) ──► event subscriber
```

**Shared fate:** if any module returns an error from `Start`, the
platform cancels the shared context, causing all other modules to stop.

## 2. `core.Module` Interface

Every module must satisfy this single interface:

```go
// libs/ogl/platform/core/app.go
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
signaled entirely via context cancellation. Modules that hold resources
(DB connections, caches) should release them inside `Start` after the
context is done, or in a `defer` inside `New`.

## 3. `platform.App` and `Run()`

```go
// libs/ogl/platform/runner.go
package platform

import (
    "context"
    "log/slog"

    oglcore "github.com/ovya/ogl/platform/core"
    "golang.org/x/sync/errgroup"
)

type App struct {
    logger  *slog.Logger
    modules []oglcore.Module
}

func New(logger *slog.Logger, modules []oglcore.Module) *App {
    return &App{logger: logger, modules: modules}
}

func (a *App) Run(ctx context.Context) error {
    g, gCtx := errgroup.WithContext(ctx)

    for _, m := range a.modules {
        g.Go(func() error {
            return m.Start(gCtx)
        })
    }

    if err := g.Wait(); err != nil {
        a.logger.Error("application stopped with error", "err", err)
        return err
    }

    a.logger.Info("application stopped gracefully")
    return nil
}
```

`Run` blocks until all modules exit. The `errgroup` context (`gCtx`) is
passed to every module. If module A fails, `errgroup` cancels `gCtx`,
which propagates to modules B and C, triggering their shutdown.

## 4. `Infrastructure` Struct Pattern

Each module defines its own `Infrastructure` struct listing the shared
resources it needs. This is the only public API surface of the module
besides `New` and `Start`.

```go
// modules/foomod/foomod.go
type Infrastructure struct {
    DBPool   *pgxpool.Pool          // shared DB pool (read/write)
    EventBus oglevents.SystemEventBus // Watermill bus wrapper
    BarSvc   defbar.BarService      // cross-module contract (injected)
    Logger   *slog.Logger
}
```

**Rules:**
- Only accept what the module actually uses — no "just in case" fields
- Cross-module dependencies come in as contract interfaces (`defbar.BarService`), never as concrete `*Module` pointers
- The DB pool is shared across modules; each module manages its own schema and migration

## 5. Module Factory: `New`

`New` is the composition root of a single module. It wires all internal
components and returns a ready-to-run `*Module`.

```go
// modules/foomod/foomod.go
func New(infra Infrastructure) (*Module, error) {
    cfg, err := config.Load(context.Background(), "")
    if err != nil {
        return nil, fmt.Errorf("foomod: load config: %w", err)
    }

    // 1. Outbound adapters
    repo := postgres.NewFooRepository(infra.DBPool)
    dispatcher := events.NewPostgresOutboxDispatcher(infra.DBPool)
    uow := ogluow.New(infra.DBPool)

    // 2. Application service (facade over command/query handlers)
    appSvc := application.NewFooApplicationService(repo, uow, dispatcher)

    // 3. Inbound adapter (Connect RPC handler)
    connectHandler := connectadapter.NewFooHandler(appSvc)

    // 4. HTTP mux + auth middleware
    mux := http.NewServeMux()
    path, handler := foov1connect.NewFooServiceHandler(
        connectHandler,
        connect.WithInterceptors(oglconnect.NewErrorLoggingInterceptor(infra.Logger)),
    )
    mux.Handle(path, connectadapter.NewAuthMiddleware(
        infra.BarSvc, infra.Logger, nil, handler,
    ))

    // 5. HTTP server (exposes /health, /debug, service routes)
    srv := oglserver.NewHTTPServer(oglserver.HTTPServerInfra{
        Config:    cfg.Server,
        Handler:   mux,
        Logger:    infra.Logger,
        HealthFns: oglserver.HealthFns{"database": appSvc.Health},
    })

    // 6. Outbox relay (polls foo.event table, publishes to EventBus)
    relay := ogloutbox.NewEnventsRelay(
        infra.DBPool, infra.EventBus, infra.Logger, "foo.event",
    )

    return &Module{relay: relay, server: srv, logger: infra.Logger}, nil
}
```

Everything inside this function is the module's private wiring. Nothing
leaks out except the `*Module` handle.

## 6. `Start()` Implementation

Each module runs its own internal `errgroup` to manage its server and
background workers:

```go
// modules/foomod/foomod.go
type Module struct {
    relay  *ogloutbox.EventsRelay
    server *oglserver.HTTPServer
    logger *slog.Logger
}

// Compile-time assertion: Module must satisfy core.Module
var _ oglcore.Module = (*Module)(nil)

func (m *Module) Start(ctx context.Context) error {
    g, gCtx := errgroup.WithContext(ctx)

    g.Go(func() error {
        return m.server.Start(gCtx) // blocks until gCtx canceled
    })

    if m.relay != nil {
        g.Go(func() error {
            m.relay.Start(gCtx) // relay exits cleanly on cancel
            return nil
        })
    }

    return g.Wait()
}
```

## 7. Composition Root Walkthrough

The composition root in `cmd/mmw/main.go`:

1. Sets up the signal context (SIGTERM/SIGINT → cancel)
2. Loads shared config
3. Opens the shared DB pool
4. Creates the Watermill raw bus and wraps it as `SystemEventBus`
5. Instantiates modules **in dependency order** — the module being depended on first
6. Wires cross-module contracts via `InprocClient`
7. Hands all modules to `platform.New().Run(ctx)`

```go
// cmd/mmw/main.go
func main() {
    ctx, cancel := signal.NotifyContext(context.Background(),
        os.Interrupt, syscall.SIGTERM)
    defer cancel()

    cfg, _ := mmwconfig.Load(ctx)
    logger, _ := oglslog.New(oglslog.HandlerText, cfg.LogLevel.SlogLevel())
    dbPool, _ := pgxpool.New(ctx, cfg.MainDatabase.URL())
    defer dbPool.Close()

    rawBus := gochannel.NewGoChannel(gochannel.Config{
        OutputChannelBuffer: 1024,
        Persistent:          true,
    }, watermill.NewSlogLogger(logger))
    defer rawBus.Close()
    systemBus := oglevents.NewWatermillBus(rawBus)

    // barmod has no cross-module dependency — instantiate first
    barModule, _ := barmod.New(barmod.Infrastructure{
        DBPool:   dbPool,
        EventBus: systemBus,
        Logger:   logger.With("module", barmod.ModuleName),
    })

    // foomod depends on barmod — wire via InprocClient
    fooModule, _ := foomod.New(foomod.Infrastructure{
        DBPool:   dbPool,
        EventBus: systemBus,
        Logger:   logger.With("module", foomod.ModuleName),
        BarSvc:   defbar.NewInprocClient(barModule), // ← in-process contract
    })

    // notifier subscribes to events from both modules
    notifEvents := append(foomod.NotifyEvents, barmod.NotifyEvents...)
    notifModule, _ := notifications.New(notifications.Infrastructure{
        Subscriber: rawBus,
        Logger:     logger.With("module", notifications.ModuleName),
        Topics:     notifEvents,
    })

    platform.New(logger, []oglcore.Module{
        barModule,
        fooModule,
        notifModule,
    }).Run(ctx) // blocks until shutdown
}
```

**`InprocClient`** wraps any implementation of the contract interface —
in the composition root that happens to be the `*Module` concrete type — so the
consumer sees only the `BarService` interface, never the concrete module.
This is defined in `contracts/definitions/barmod/inproc_client.go`.

## 8. Compile-Time Safety

Each module file includes this assertion immediately after the struct
definition:

```go
var _ oglcore.Module = (*Module)(nil)
```

If `Start(ctx context.Context) error` is missing or has the wrong
signature, the build fails. This is the first line of architecture
enforcement — no runtime surprises.

## 9. Adding a New Module — Checklist

- [ ] Create `modules/newmod/newmod.go` with `Infrastructure`, `Module`, `New`, `Start`, and the compile-time assertion
- [ ] Define `contracts/definitions/newmodsvc/api.go`, `dto.go`, `errors.go`, `inproc_client.go` if other modules will call it
- [ ] Add `modules/newmod` to `go.work`
- [ ] Add `modules/newmod` to `cmd/mmw/main.go`: instantiate, wire via `InprocClient` if needed, append to `modules` slice
- [ ] Add `newmod.NotifyEvents` to the notifier's event list if applicable
- [ ] Add DB migrations under `modules/newmod/internal/infra/persistence/migrations/`
- [ ] Verify `go build ./cmd/mmw/...` compiles cleanly
