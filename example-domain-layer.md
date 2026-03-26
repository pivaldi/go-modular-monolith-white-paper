# Domain Layer Examples

The domain layer contains the core business logic. It has **zero external
dependencies** — no database drivers, no HTTP frameworks, no logging.

## Aggregate Root

The aggregate root enforces all business invariants. Fields are private;
state changes happen only through methods that validate and emit events.

```go
// modules/foosvc/internal/domain/foo.go
package domain

import (
    "time"
    "github.com/google/uuid"
)

// Foo is the aggregate root. All state changes go through its methods.
type Foo struct {
    id          FooID
    title       FooTitle
    status      FooStatus
    priority    Priority
    ownerID     uuid.UUID
    createdAt   time.Time
    updatedAt   time.Time
    events      []DomainEvent
}

// NewFoo creates a new Foo, validates invariants, and emits FooCreated.
func NewFoo(title FooTitle, priority Priority, ownerID uuid.UUID) (*Foo, error) {
    if ownerID == uuid.Nil {
        return nil, ErrOwnerRequired
    }
    now := time.Now().UTC()
    f := &Foo{
        id:        NewFooID(),
        title:     title,
        status:    StatusPending,
        priority:  priority,
        ownerID:   ownerID,
        createdAt: now,
        updatedAt: now,
    }
    f.events = append(f.events, FooCreated{FooID: f.id, OwnerID: ownerID, At: now})
    return f, nil
}

// Complete transitions the Foo to completed status and emits FooCompleted.
func (f *Foo) Complete() error {
    if f.status == StatusCompleted {
        return ErrAlreadyCompleted
    }
    if f.status == StatusCancelled {
        return ErrCancelledCannotComplete
    }
    f.status = StatusCompleted
    f.updatedAt = time.Now().UTC()
    f.events = append(f.events, FooCompleted{FooID: f.id, At: f.updatedAt})
    return nil
}

// PopEvents returns accumulated domain events and clears the internal list.
func (f *Foo) PopEvents() []DomainEvent {
    evts := f.events
    f.events = nil
    return evts
}

// Getters (no setters — mutations go through methods only)
func (f *Foo) ID() FooID         { return f.id }
func (f *Foo) Title() FooTitle   { return f.title }
func (f *Foo) Status() FooStatus { return f.status }
func (f *Foo) OwnerID() uuid.UUID { return f.ownerID }
```

## Value Objects

Value objects are distinct named types. Validation lives in their
constructors, not scattered across the application layer.

```go
// modules/foosvc/internal/domain/value_objects.go
package domain

import "github.com/google/uuid"

// FooID wraps uuid.UUID so the compiler catches id mix-ups.
type FooID struct{ v uuid.UUID }

func NewFooID() FooID { return FooID{v: uuid.New()} }
func FooIDFromString(s string) (FooID, error) {
    id, err := uuid.Parse(s)
    if err != nil {
        return FooID{}, ErrInvalidFooID
    }
    return FooID{v: id}, nil
}
func (id FooID) String() string  { return id.v.String() }
func (id FooID) UUID() uuid.UUID { return id.v }

// FooTitle is a validated non-empty string with a max length.
type FooTitle struct{ v string }

func NewFooTitle(s string) (FooTitle, error) {
    if len(s) == 0 {
        return FooTitle{}, ErrTitleEmpty
    }
    if len(s) > 200 {
        return FooTitle{}, ErrTitleTooLong
    }
    return FooTitle{v: s}, nil
}
func (t FooTitle) String() string { return t.v }

// FooStatus is an enum.
type FooStatus string

const (
    StatusPending   FooStatus = "pending"
    StatusActive    FooStatus = "active"
    StatusCompleted FooStatus = "completed"
    StatusCancelled FooStatus = "cancelled"
)

// Priority is an enum with an ordering.
type Priority string

const (
    PriorityLow    Priority = "low"
    PriorityMedium Priority = "medium"
    PriorityHigh   Priority = "high"
    PriorityUrgent Priority = "urgent"
)
```

## Domain Events

Events are plain structs. They carry the minimum data needed for
consumers. All events implement `DomainEvent`.

```go
// modules/foosvc/internal/domain/events.go
package domain

import (
    "time"
    "github.com/google/uuid"
)

// DomainEvent is the marker interface for all foo domain events.
type DomainEvent interface {
    EventType() string
}

// AllEvents lists the event type strings this module publishes.
// The composition root uses this to configure the notifier module.
var AllEvents = []string{
    "foo.created",
    "foo.completed",
    "foo.cancelled",
    "foo.updated",
}

type FooCreated struct {
    FooID   FooID
    OwnerID uuid.UUID
    At      time.Time
}
func (e FooCreated) EventType() string { return "foo.created" }

type FooCompleted struct {
    FooID FooID
    At    time.Time
}
func (e FooCompleted) EventType() string { return "foo.completed" }

type FooUpdated struct {
    FooID FooID
    At    time.Time
}
func (e FooUpdated) EventType() string { return "foo.updated" }
```

## Snapshot Pattern

The repository does not access private fields directly. The aggregate
exposes a `Snapshot` for persistence and a `FromSnapshot` constructor
for reconstitution:

```go
// modules/foosvc/internal/domain/snapshot.go
package domain

import "time"

// FooSnapshot is a plain-data representation used by the repository adapter.
type FooSnapshot struct {
    ID        string
    Title     string
    Status    string
    Priority  string
    OwnerID   string
    CreatedAt time.Time
    UpdatedAt time.Time
}

// ToSnapshot produces a persistence-ready snapshot (no domain logic).
func (f *Foo) ToSnapshot() FooSnapshot {
    return FooSnapshot{
        ID:        f.id.String(),
        Title:     f.title.String(),
        Status:    string(f.status),
        Priority:  string(f.priority),
        OwnerID:   f.ownerID.String(),
        CreatedAt: f.createdAt,
        UpdatedAt: f.updatedAt,
    }
}

// FromSnapshot reconstitutes a Foo from a snapshot without emitting events.
func FromSnapshot(s FooSnapshot) (*Foo, error) {
    id, err := FooIDFromString(s.ID)
    if err != nil {
        return nil, err
    }
    title, err := NewFooTitle(s.Title)
    if err != nil {
        return nil, err
    }
    ownerID, err := uuid.Parse(s.OwnerID)
    if err != nil {
        return nil, err
    }
    return &Foo{
        id:        id,
        title:     title,
        status:    FooStatus(s.Status),
        priority:  Priority(s.Priority),
        ownerID:   ownerID,
        createdAt: s.CreatedAt,
        updatedAt: s.UpdatedAt,
    }, nil
}
```

## Domain Errors

```go
// modules/foosvc/internal/domain/errors.go
package domain

import "errors"

var (
    ErrOwnerRequired           = errors.New("owner is required")
    ErrTitleEmpty              = errors.New("title cannot be empty")
    ErrTitleTooLong            = errors.New("title exceeds 200 characters")
    ErrInvalidFooID            = errors.New("invalid foo id")
    ErrAlreadyCompleted        = errors.New("foo is already completed")
    ErrCancelledCannotComplete = errors.New("cancelled foo cannot be completed")
    ErrNotFound                = errors.New("foo not found")
)
```
