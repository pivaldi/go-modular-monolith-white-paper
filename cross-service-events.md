# Cross-Service Events

Modules communicate asynchronously via domain events. The outbox
pattern guarantees events are never lost even if the process crashes
between a DB commit and a publish.

## Architecture Overview

```
foosvc (producer)                   barsvc / notifier (consumers)
─────────────────                   ────────────────────────────
Command handler                     EventsRelay (background)
  └── uow.Do(ctx, fn)                 └── poll foo.event table
      ├── repo.Save()                 └── publish to Watermill bus
      └── dispatcher.Dispatch()           └── barsvc subscriber
          └── INSERT foo.event                 └── handle event
          (same transaction)
```

Events are written atomically with the business data. If the process
crashes before the relay polls, events are still in the DB and will be
published on the next restart.

## Step 1: Declare Events in the Domain

```go
// modules/foosvc/internal/domain/events.go
package domain

var AllEvents = []string{
    "foo.created",
    "foo.completed",
    "foo.deleted",
}

type FooCreated struct {
    FooID   string
    OwnerID string
}
func (e FooCreated) EventType() string { return "foo.created" }
```

Export `AllEvents` from the module root for the composition root to use:

```go
// modules/foosvc/foosvc.go
var NotifyEvents = domain.AllEvents
```

## Step 2: Write Events to the Outbox

The `PostgresOutboxDispatcher` (see [example-outbound-adapter.md](example-outbound-adapter.md))
writes events to `foo.event` inside the Unit of Work transaction.
No direct Watermill calls happen in the command handler.

```
BEGIN TRANSACTION
  INSERT INTO foo.foo  (business data)
  INSERT INTO foo.event (domain events)
COMMIT
```

## Step 3: Relay Events to Watermill

The `EventsRelay` (from `ogl/db/outbox`) is started as a goroutine inside
`Module.Start`. It polls the outbox table and publishes unpublished events:

```go
// Inside Module.Start():
g.Go(func() error {
    relay.Start(gCtx) // blocks until gCtx is canceled
    return nil
})
```

The relay is created in `New` with the table name and event bus:

```go
relay := ogloutbox.NewEnventsRelay(
    infra.DBPool,
    infra.EventBus,    // Watermill SystemEventBus wrapper
    infra.Logger,
    "foo.event",       // outbox table name (schema.table)
)
```

## Step 4: Subscribe in a Consuming Module

Consuming modules receive the raw Watermill bus (not the wrapped bus)
and subscribe to topics matching the producer's `AllEvents`:

```go
// modules/notifications/notifications.go
type Infrastructure struct {
    Subscriber  *gochannel.GoChannel   // raw Watermill bus
    Logger      *slog.Logger
    Topics      []string               // from foosvc.NotifyEvents + barsvc.NotifyEvents
    WithNotifier bool
}
```

In `cmd/mmw/main.go`, the topics are assembled from all producers:

```go
notifEvents := append(foosvc.NotifyEvents, barsvc.NotifyEvents...)
notifModule, _ := notifications.New(notifications.Infrastructure{
    Subscriber: rawBus,
    Logger:     logger.With("module", notifications.ModuleName),
    Topics:     notifEvents,
})
```

## Outbox Table Schema

Each module manages its own outbox table, scoped to its schema:

```sql
-- modules/foosvc/internal/infra/persistence/migrations/002_create_event_table.sql
CREATE TABLE foo.event (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_id UUID NOT NULL,
    event_type   VARCHAR(100) NOT NULL,
    payload      JSONB NOT NULL,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    published_at TIMESTAMPTZ
);

CREATE INDEX ON foo.event (published_at) WHERE published_at IS NULL;
```

## Event Bus in the Composition Root

The raw Watermill bus and the wrapped `SystemEventBus` are created once
and shared. Producers receive `SystemEventBus` (write side); consumers
that need raw subscriptions receive the `*gochannel.GoChannel` directly:

```go
rawBus := gochannel.NewGoChannel(
    gochannel.Config{OutputChannelBuffer: 1024, Persistent: true},
    watermill.NewSlogLogger(logger),
)
systemBus := oglevents.NewWatermillBus(rawBus)

// foosvc gets the wrapped bus (for the relay)
foosvc.New(foosvc.Infrastructure{EventBus: systemBus, ...})

// notifications gets the raw bus (for subscriptions)
notifications.New(notifications.Infrastructure{Subscriber: rawBus, ...})
```
