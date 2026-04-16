# Outbound Adapter Examples

Outbound adapters (secondary adapters) implement the port interfaces
defined in the application layer. They translate between domain types
and external systems (PostgreSQL, event bus).

## Postgres Repository

The repository receives a `*pfuow.UnitOfWork` — not a raw `*pgxpool.Pool`.
`uow.Executor(ctx)` returns the active transaction when inside
`uow.WithTransaction`, or falls back to the pool's connection otherwise.
This makes the repository transaction-aware without knowing about transactions
directly.

Snapshots are persisted via `pgx.NamedArgs(pfdb.StructArgs(snap))`, which
reflects `db`-tagged struct fields into a `map[string]any` compatible with pgx.

```go
// modules/todo/internal/adapters/outbound/persistence/postgres/todo_repository.go
package postgres

import (
    "context"
    "errors"

    "github.com/google/uuid"
    "github.com/jackc/pgx/v5"
    pfdb  "github.com/piprim/mmw/pkg/platform/db"
    pfuow "github.com/piprim/mmw/pkg/platform/pg/uow"
    "github.com/rotisserie/eris"

    "github.com/pivaldi/mmw-todo/internal/application/ports"
    "github.com/pivaldi/mmw-todo/internal/domain"
)

// PostgresTodoRepository implements ports.TodoRepository.
type PostgresTodoRepository struct {
    uow *pfuow.UnitOfWork
}

// Compile-time assertion
var _ ports.TodoRepository = (*PostgresTodoRepository)(nil)

func NewPostgresTodoRepository(uow *pfuow.UnitOfWork) *PostgresTodoRepository {
    return &PostgresTodoRepository{uow: uow}
}

// Save persists a new todo using named args derived from the snapshot.
func (r *PostgresTodoRepository) Save(ctx context.Context, todo *domain.Todo) error {
    query := `
        INSERT INTO todo.todo (id, title, description, status, priority, due_date,
                               created_at, updated_at, completed_at, user_id)
        VALUES (@id, @title, @description, @status, @priority, @due_date,
                @created_at, @updated_at, @completed_at, @user_id)
    `
    _, err := r.uow.Executor(ctx).Exec(ctx, query, pgx.NamedArgs(pfdb.StructArgs(todo.Snapshot())))
    if err != nil {
        return eris.Wrap(err, "saving todo")
    }
    return nil
}

// FindByID retrieves a todo scoped to the given user.
// pgx.RowToStructByName scans columns directly into TodoSnapshot via db tags.
func (r *PostgresTodoRepository) FindByID(ctx context.Context, id domain.TodoID, userID uuid.UUID) (*domain.Todo, error) {
    query := `
        SELECT id, title, description, status, priority, due_date,
               created_at, updated_at, completed_at, user_id
        FROM todo.todo
        WHERE id = $1 AND user_id = $2
    `
    rows, err := r.uow.Executor(ctx).Query(ctx, query, id.String(), userID)
    if err != nil {
        return nil, eris.Wrap(err, "querying todo")
    }

    snap, err := pgx.CollectOneRow(rows, pgx.RowToStructByName[domain.TodoSnapshot])
    if err != nil {
        if errors.Is(err, pgx.ErrNoRows) {
            return nil, domain.ErrTodoNotFound
        }
        return nil, eris.Wrap(err, "collecting todo")
    }

    return domain.ReconstituteTodo(&snap), nil
}

// Update persists changes to an existing todo.
func (r *PostgresTodoRepository) Update(ctx context.Context, todo *domain.Todo) error {
    query := `
        UPDATE todo.todo
        SET title        = @title,
            description  = @description,
            status       = @status,
            priority     = @priority,
            due_date     = @due_date,
            updated_at   = @updated_at,
            completed_at = @completed_at
        WHERE id = @id AND user_id = @user_id
    `
    result, err := r.uow.Executor(ctx).Exec(ctx, query, pgx.NamedArgs(pfdb.StructArgs(todo.Snapshot())))
    if err != nil {
        return eris.Wrap(err, "updating todo")
    }
    if result.RowsAffected() == 0 {
        return domain.ErrTodoNotFound
    }
    return nil
}

// Delete removes a todo scoped to the given user.
func (r *PostgresTodoRepository) Delete(ctx context.Context, id domain.TodoID, userID uuid.UUID) error {
    result, err := r.uow.Executor(ctx).Exec(ctx,
        `DELETE FROM todo.todo WHERE id = $1 AND user_id = $2`,
        id.String(), userID.String(),
    )
    if err != nil {
        return eris.Wrap(err, "deleting todo")
    }
    if result.RowsAffected() == 0 {
        return domain.ErrTodoNotFound
    }
    return nil
}

func (r *PostgresTodoRepository) Health(ctx context.Context) (any, error) {
    row := r.uow.Executor(ctx).QueryRow(ctx, "SELECT count(*) FROM todo.todo")
    var count int
    if err := row.Scan(&count); err != nil {
        return 0, eris.Wrap(err, "scan row")
    }
    return count, nil
}
```

**`uow.Executor(ctx)`** returns the pgx executor appropriate for the current
context — a `pgx.Tx` if a transaction is active, or a `*pgxpool.Pool` connection
otherwise. This allows the repository to be transaction-aware without knowing
about transactions directly.

## Outbox Event Dispatcher

Events are not published directly to Watermill. They are written to a
`todo.event` outbox table **in the same transaction** as the domain write.
A background `EventsRelay` then polls and publishes them.

The dispatcher uses `domainTopics` — a package-level map — to translate
semantic domain event type strings into Watermill routing keys (topic constants
from `contracts/go/application/todo/events_gen.go`).

```go
// modules/todo/internal/adapters/outbound/events/outbox_dispatcher.go
package events

import (
    "context"
    "encoding/json"

    "github.com/jackc/pgx/v5"
    pfuow "github.com/piprim/mmw/pkg/platform/pg/uow"
    "github.com/pivaldi/mmw-todo/internal/domain"
    "github.com/rotisserie/eris"
    "google.golang.org/protobuf/encoding/protojson"
)

type PostgresOutboxDispatcher struct {
    uow *pfuow.UnitOfWork
}

func NewPostgresOutboxDispatcher(uow *pfuow.UnitOfWork) *PostgresOutboxDispatcher {
    return &PostgresOutboxDispatcher{uow: uow}
}

// Dispatch saves all events to the outbox table in a single batch.
// The stored event_type is the Watermill routing key (from domainTopics).
func (d *PostgresOutboxDispatcher) Dispatch(ctx context.Context, events []domain.DomainEvent) error {
    if len(events) == 0 {
        return nil
    }

    batch := &pgx.Batch{}
    query := `INSERT INTO todo.event (event_type, payload, occurred_at) VALUES ($1, $2::jsonb, $3)`

    for _, event := range events {
        topic, ok := domainTopics[event.EventType()]
        if !ok {
            return eris.Errorf("no routing key for domain event type %q", event.EventType())
        }

        var (
            payload []byte
            err     error
        )
        // Prefer protojson encoding for known event types; fallback to json.Marshal
        if protoMsg := domainToProto(event); protoMsg != nil {
            payload, err = protojson.Marshal(protoMsg)
        } else {
            payload, err = json.Marshal(event)
        }
        if err != nil {
            return eris.Wrapf(err, "failed to marshal event %s", event.EventType())
        }

        batch.Queue(query, topic, string(payload), event.GetOccurredAt())
    }

    br := d.uow.Executor(ctx).SendBatch(ctx, batch)
    defer br.Close()

    for i := range events {
        if _, err := br.Exec(); err != nil {
            return eris.Wrapf(err, "failed to insert outbox event at index %d", i)
        }
    }
    return nil
}
```

The `domainTopics` map is the single place where domain semantics are
translated to transport concerns:

```go
// modules/todo/internal/adapters/outbound/events/topics.go
package events

import (
    tododef "github.com/pivaldi/mmw-contracts/go/application/todo"
    "github.com/pivaldi/mmw-todo/internal/domain"
)

// domainTopics maps semantic domain event types to Watermill routing keys.
var domainTopics = map[string]string{
    domain.EventTypeCreated:          tododef.TopicUserTaskCreated,
    domain.EventTypeUpdated:          tododef.TopicUserTaskUpdated,
    domain.EventTypeCompleted:        tododef.TopicUserTaskCompleted,
    domain.EventTypeReopened:         tododef.TopicUserTaskReopened,
    domain.EventTypeDeleted:          tododef.TopicUserTaskDeleted,
    domain.EventTypeUserTasksDeleted: tododef.TopicUserTasksDeleted,
}
```

## Why the Outbox Pattern?

If events were published directly to Watermill after the DB commit,
a crash between the commit and the publish would silently lose events.
The outbox pattern writes events atomically with the business data.

```
Command handler calls uow.WithTransaction(ctx, fn):
  └── BEGIN TRANSACTION
      ├── repo.Save(ctx, todo)            → INSERT INTO todo.todo
      └── dispatcher.Dispatch(ctx, evts)  → INSERT INTO todo.event
  └── COMMIT (both or neither)

Background EventsRelay (started in Module.Start):
  └── SELECT * FROM todo.event WHERE published_at IS NULL
      └── Publishes each row to Watermill SystemEventBus
          └── UPDATE todo.event SET published_at = now()
```

The `EventsRelay` is provided by `mmw/pkg/platform/db/outbox` and started
in the module's `Start()` method alongside the HTTP server and event router.
