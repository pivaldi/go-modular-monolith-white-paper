# Cross-Service Events

Modules communicate asynchronously via domain events. The outbox pattern guarantees
events are never lost even if the process crashes between a DB commit and a publish.

## Architecture Overview

```
auth module (producer)                  todo / notifications (consumers)
──────────────────────                  ─────────────────────────────────
Command handler                         EventsRelay (background)
  └── uow.WithTransaction(ctx, fn)        └── poll auth.event table
      ├── repo.Save()                     └── publish to Watermill bus
      └── dispatcher.Dispatch()               └── notifications subscriber
          └── INSERT auth.event                   └── handle event
          (same transaction)
```

Events are written atomically with the business data. If the process crashes before
the relay polls, events are still in the DB and will be published on the next restart.

## Step 1: Declare Event Topics in the Contract

Topic constants and event type aliases are generated into `contracts/go/application/{domain}/events_gen.go`
by `protoc-gen-go-contracts`. They come from proto messages annotated with `(options.v1.topic)`.

```go
// contracts/go/application/auth/events_gen.go (generated — do not edit)
const (
    TopicUserRegistered  = "auth.user.registered.v1"
    TopicUserDeleted     = "auth.user.deleted.v1"
    TopicPasswordChanged = "auth.user.password_changed.v1"
    TopicUserLoggedIn    = "auth.user.logged_in.v1"
)

var Topics = []string{TopicUserRegistered, TopicUserDeleted, TopicPasswordChanged, TopicUserLoggedIn}
```

The `Topics` slice is what the composition root uses to subscribe the notifications module.

## Step 2: Emit Domain Events in Command Handlers

```go
// modules/auth/internal/application/command/register_user.go
func (c *RegisterUserCommand) Execute(ctx context.Context, req *authv1.RegisterRequest) (*authv1.RegisterResponse, error) {
    var resp *authv1.RegisterResponse
    err := c.unitOfWork.WithTransaction(ctx, func(txCtx context.Context) error {
        user, err := domain.NewUser(email, password)
        if err != nil { return err }

        if err := c.repository.Save(txCtx, user); err != nil { return err }

        // Dispatch into the outbox table (same transaction)
        return c.eventDispatcher.Dispatch(txCtx, user.Events())
    })
    return resp, err
}
```

## Step 3: Write Events to the Outbox

The `PostgresOutboxDispatcher` writes to `auth.event` **inside the Unit of Work transaction**.
No direct Watermill calls happen in the command handler.

```
BEGIN TRANSACTION
  INSERT INTO auth.user  (business data)
  INSERT INTO auth.event (domain events)
COMMIT
```

## Step 4: Relay Events to Watermill

The `EventsRelay` (from `mmw/pkg/platform/db/outbox`) is started as a goroutine in
`Module.Start`. It polls the outbox table and publishes unpublished events:

```go
// modules/auth/auth.go — inside New()
relay := pfoutbox.NewEnventsRelay(
    infra.DBPool,
    infra.EventBus,     // SystemEventBus interface (write side)
    infra.Logger,
    "auth.event",       // outbox table name (schema.table)
)
```

```go
// Inside Module.Start():
g.Go(func() error {
    m.relay.Start(gCtx) // blocks until gCtx is canceled
    return nil
})
```

## Step 5: Subscribe in the Notifications Module

The notifications module receives the raw Watermill `GoChannel` and subscribes
to topics from the generated `Topics` slices:

```go
// cmd/mmw/main.go
notifModule, _ := notifications.New(notifications.Infrastructure{
    Subscriber: rawBus,
    Logger:     logger.With("module", notifications.ModuleName),
    Topics:     append(tododef.Topics, authdef.Topics...),
    // tododef.Topics = ["todo.userTasks.created.v1", "todo.userTask.deleted.v1", …]
    // authdef.Topics = ["auth.user.registered.v1", "auth.user.deleted.v1", …]
})
```

For targeted inbound event handling (e.g. todo reacts to auth.user.deleted), a
Watermill router is set up in `newEventRouter`:

```go
// modules/todo/todo.go
router.AddConsumerHandler(
    "todo.on_auth_user_deleted",     // unique handler name (stable across restarts)
    defauth.TopicUserDeleted,        // = "auth.user.deleted.v1"
    infra.Subscriber,
    inevents.HandleUserDeleted(deleteUserTasksCmd),
)
```

## Outbox Table Schema

Each module manages its own outbox table, scoped to its PostgreSQL schema:

```sql
-- modules/auth/internal/infra/persistence/migrations/scripts/
CREATE TABLE auth.event (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_id UUID NOT NULL,
    event_type   VARCHAR(100) NOT NULL,
    payload      BYTEA NOT NULL,           -- protobuf-encoded
    occurred_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    published_at TIMESTAMPTZ
);

CREATE INDEX ON auth.event (published_at) WHERE published_at IS NULL;
```

## Event Bus in the Composition Root

The raw Watermill bus and the wrapped `SystemEventBus` are created once and shared.
Producers receive `SystemEventBus` (write side via relay); consumers that need raw
subscriptions receive the `*gochannel.GoChannel` directly:

```go
rawBus := gochannel.NewGoChannel(
    gochannel.Config{OutputChannelBuffer: 1024, Persistent: true},
    watermill.NewSlogLogger(logger),
)
eventBus := pfevents.NewWatermillBus(rawBus)

// Auth gets the wrapped bus (for the outbox relay to publish on)
auth.New(auth.Infrastructure{EventBus: eventBus, ...})

// Todo gets both (relay publishes; event router subscribes)
todo.New(todo.Infrastructure{EventBus: eventBus, Subscriber: rawBus, ...})

// Notifications gets the raw bus (for subscriptions only)
notifications.New(notifications.Infrastructure{Subscriber: rawBus, ...})
```

## Outbox Guarantees

| Property | Detail |
|---|---|
| At-least-once delivery | A crash between `Publish` and the `UPDATE` causes relay to retry on next tick. Consumers must be idempotent. |
| Ordering | Rows fetched `ORDER BY occurred_at ASC` — preserves intra-aggregate event order within a batch. |
| Latency | Up to 2 s between domain write and bus publication (relay tick interval). |
| Multi-instance safe | `SELECT … FOR UPDATE SKIP LOCKED` prevents two relay instances from processing the same row. |
