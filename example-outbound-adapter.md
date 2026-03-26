# Outbound Adapter Examples

Outbound adapters (secondary adapters) implement the port interfaces
defined in the application layer. They translate between domain types
and external systems (PostgreSQL, event bus).

## Postgres Repository

The repository receives a `*pgxpool.Pool` directly — no ORM. It uses
the domain's `ToSnapshot()` / `FromSnapshot()` for persistence.

```go
// modules/foomod/internal/adapters/outbound/persistence/postgres/foo_repository.go
package postgres

import (
    "context"
    "errors"

    "github.com/jackc/pgx/v5"
    "github.com/jackc/pgx/v5/pgxpool"
    ogluow "github.com/ovya/ogl/pg/uow"
    "github.com/example/mmw-foomod/internal/application/ports"
    "github.com/example/mmw-foomod/internal/domain"
)

// PostgresFooRepository implements ports.FooRepository.
type PostgresFooRepository struct {
    pool *pgxpool.Pool
}

// Compile-time assertion
var _ ports.FooRepository = (*PostgresFooRepository)(nil)

func NewFooRepository(pool *pgxpool.Pool) *PostgresFooRepository {
    return &PostgresFooRepository{pool: pool}
}

func (r *PostgresFooRepository) Save(ctx context.Context, foo *domain.Foo) error {
    snap := foo.ToSnapshot()
    exec := ogluow.GetExecutor(ctx, r.pool) // uses tx if one is active
    _, err := exec.Exec(ctx,
        `INSERT INTO foo.foo (id, title, status, priority, owner_id, created_at, updated_at)
         VALUES ($1, $2, $3, $4, $5, $6, $7)`,
        snap.ID, snap.Title, snap.Status, snap.Priority, snap.OwnerID,
        snap.CreatedAt, snap.UpdatedAt,
    )
    return err
}

func (r *PostgresFooRepository) FindByID(ctx context.Context, id domain.FooID) (*domain.Foo, error) {
    exec := ogluow.GetExecutor(ctx, r.pool)
    row := exec.QueryRow(ctx,
        `SELECT id, title, status, priority, owner_id, created_at, updated_at
         FROM foo.foo WHERE id = $1`,
        id.String(),
    )

    var snap domain.FooSnapshot
    err := row.Scan(&snap.ID, &snap.Title, &snap.Status, &snap.Priority,
        &snap.OwnerID, &snap.CreatedAt, &snap.UpdatedAt)
    if errors.Is(err, pgx.ErrNoRows) {
        return nil, domain.ErrNotFound
    }
    if err != nil {
        return nil, err
    }

    return domain.FromSnapshot(snap)
}

func (r *PostgresFooRepository) Delete(ctx context.Context, id domain.FooID) error {
    exec := ogluow.GetExecutor(ctx, r.pool)
    _, err := exec.Exec(ctx, `DELETE FROM foo.foo WHERE id = $1`, id.String())
    return err
}

func (r *PostgresFooRepository) Health(ctx context.Context) error {
    return r.pool.Ping(ctx)
}
```

**`ogluow.GetExecutor`** returns the transaction executor if a UoW
transaction is active on the context, or falls back to the pool. This
allows the repository to be transaction-aware without knowing about
transactions directly.

## Outbox Event Dispatcher

Events are not published directly to the event bus. They are written
to a `foo.event` outbox table **in the same transaction** as the
business operation. A background relay then publishes them to Watermill.

```go
// modules/foomod/internal/adapters/outbound/events/outbox_dispatcher.go
package events

import (
    "context"
    "encoding/json"

    "github.com/jackc/pgx/v5/pgxpool"
    ogluow "github.com/ovya/ogl/pg/uow"
    "github.com/example/mmw-foomod/internal/application/ports"
    "github.com/example/mmw-foomod/internal/domain"
)

// PostgresOutboxDispatcher implements ports.EventDispatcher.
// It writes domain events to the outbox table inside the active transaction.
type PostgresOutboxDispatcher struct {
    pool *pgxpool.Pool
}

var _ ports.EventDispatcher = (*PostgresOutboxDispatcher)(nil)

func NewPostgresOutboxDispatcher(pool *pgxpool.Pool) *PostgresOutboxDispatcher {
    return &PostgresOutboxDispatcher{pool: pool}
}

func (d *PostgresOutboxDispatcher) Dispatch(ctx context.Context, events []domain.DomainEvent) error {
    if len(events) == 0 {
        return nil
    }

    exec := ogluow.GetExecutor(ctx, d.pool) // uses the same tx as the repo

    for _, evt := range events {
        payload, err := json.Marshal(evt)
        if err != nil {
            return err
        }
        // aggregateID is a package-level helper that extracts the aggregate ID
        // string from a domain.DomainEvent (e.g. FooCreated.FooID.String()).
        _, err = exec.Exec(ctx,
            `INSERT INTO foo.event (aggregate_id, event_type, payload)
             VALUES ($1, $2, $3)`,
            aggregateID(evt), evt.EventType(), payload,
        )
        if err != nil {
            return err
        }
    }
    return nil
}
```

## Why the Outbox Pattern?

If events were published directly to Watermill after the DB commit,
a crash between the commit and the publish would silently lose events.
The outbox pattern writes events atomically with the business data.

```
Command handler calls uow.Do(ctx, fn):
  └── BEGIN TRANSACTION
      ├── repo.Save(ctx, foo)          → INSERT INTO foo.foo
      └── dispatcher.Dispatch(ctx, evts) → INSERT INTO foo.event
  └── COMMIT (both or neither)

Background EventsRelay (started in Module.Start):
  └── Polls foo.event WHERE published_at IS NULL
      └── Publishes to Watermill SystemEventBus
          └── UPDATE foo.event SET published_at = now()
```

The `EventsRelay` is provided by `ogl/db/outbox` and started in the
module's `Start()` method alongside the HTTP server.
