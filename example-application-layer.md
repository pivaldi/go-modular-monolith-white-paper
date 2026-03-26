# Application Layer Examples

The application layer orchestrates use cases. It has no business logic
(that lives in the domain) and no infrastructure knowledge (that lives
in adapters). It depends only on domain types and port interfaces.

## Ports (Secondary Interfaces)

Ports are interfaces the application layer defines. Adapters implement them.

```go
// modules/foosvc/internal/application/ports/repository.go
package ports

import (
    "context"
    "github.com/example/mmw-foosvc/internal/domain"
)

type FooRepository interface {
    Save(ctx context.Context, foo *domain.Foo) error
    FindByID(ctx context.Context, id domain.FooID) (*domain.Foo, error)
    FindAll(ctx context.Context, ownerID string) ([]*domain.Foo, error)
    Delete(ctx context.Context, id domain.FooID) error
    Health(ctx context.Context) error
}

// modules/foosvc/internal/application/ports/events.go
type EventDispatcher interface {
    Dispatch(ctx context.Context, events []domain.DomainEvent) error
}

// modules/foosvc/internal/application/ports/uow.go
// UnitOfWork wraps a command in a database transaction.
type UnitOfWork interface {
    Do(ctx context.Context, fn func(ctx context.Context) error) error
}
```

## Application Service Facade

The `FooApplicationService` is a thin facade that delegates to
command and query handlers. It is the only type the inbound adapter
(Connect handler) depends on.

```go
// modules/foosvc/internal/application/service.go
package application

import (
    "context"
    "github.com/example/mmw-foosvc/internal/application/command"
    "github.com/example/mmw-foosvc/internal/application/dto"
    "github.com/example/mmw-foosvc/internal/application/ports"
    "github.com/example/mmw-foosvc/internal/application/query"
)

type FooApplicationService struct {
    createHandler   *command.CreateFooHandler
    completeHandler *command.CompleteFooHandler
    deleteHandler   *command.DeleteFooHandler
    getFooHandler   *query.GetFooHandler
    listHandler     *query.ListFoosHandler
    repo            ports.FooRepository
}

func NewFooApplicationService(
    repo ports.FooRepository,
    uow ports.UnitOfWork,
    dispatcher ports.EventDispatcher,
) *FooApplicationService {
    return &FooApplicationService{
        createHandler:   command.NewCreateFooHandler(repo, uow, dispatcher),
        completeHandler: command.NewCompleteFooHandler(repo, uow, dispatcher),
        deleteHandler:   command.NewDeleteFooHandler(repo, uow),
        getFooHandler:   query.NewGetFooHandler(repo),
        listHandler:     query.NewListFoosHandler(repo),
        repo:            repo,
    }
}

func (s *FooApplicationService) CreateFoo(ctx context.Context, cmd dto.CreateFooCommand) (*dto.FooDTO, error) {
    return s.createHandler.Handle(ctx, cmd)
}

func (s *FooApplicationService) CompleteFoo(ctx context.Context, cmd dto.CompleteFooCommand) error {
    return s.completeHandler.Handle(ctx, cmd)
}

func (s *FooApplicationService) GetFoo(ctx context.Context, q dto.GetFooQuery) (*dto.FooDTO, error) {
    return s.getFooHandler.Handle(ctx, q)
}

func (s *FooApplicationService) ListFoos(ctx context.Context, q dto.ListFoosQuery) ([]*dto.FooDTO, error) {
    return s.listHandler.Handle(ctx, q)
}

func (s *FooApplicationService) Health(ctx context.Context) error {
    return s.repo.Health(ctx)
}
```

## Command Handler (Write Side)

Command handlers perform writes. They use the Unit of Work to wrap the
operation in a transaction, then dispatch domain events.

```go
// modules/foosvc/internal/application/command/create_foo.go
package command

import (
    "context"
    "fmt"

    "github.com/example/mmw-foosvc/internal/application/dto"
    "github.com/example/mmw-foosvc/internal/application/ports"
    "github.com/example/mmw-foosvc/internal/domain"
)

type CreateFooHandler struct {
    repo       ports.FooRepository
    uow        ports.UnitOfWork
    dispatcher ports.EventDispatcher
}

func NewCreateFooHandler(repo ports.FooRepository, uow ports.UnitOfWork, dispatcher ports.EventDispatcher) *CreateFooHandler {
    return &CreateFooHandler{repo: repo, uow: uow, dispatcher: dispatcher}
}

func (h *CreateFooHandler) Handle(ctx context.Context, cmd dto.CreateFooCommand) (*dto.FooDTO, error) {
    title, err := domain.NewFooTitle(cmd.Title)
    if err != nil {
        return nil, fmt.Errorf("create foo: %w", err)
    }

    foo, err := domain.NewFoo(title, domain.Priority(cmd.Priority), cmd.OwnerID)
    if err != nil {
        return nil, fmt.Errorf("create foo: %w", err)
    }

    if err := h.uow.Do(ctx, func(ctx context.Context) error {
        if err := h.repo.Save(ctx, foo); err != nil {
            return err
        }
        return h.dispatcher.Dispatch(ctx, foo.PopEvents())
    }); err != nil {
        return nil, err
    }

    return dto.FooDTOFromDomain(foo), nil
}
```

## Query Handler (Read Side)

Query handlers perform reads only — no writes, no events, no Unit of Work.

```go
// modules/foosvc/internal/application/query/get_foo.go
package query

import (
    "context"
    "fmt"

    "github.com/example/mmw-foosvc/internal/application/dto"
    "github.com/example/mmw-foosvc/internal/application/ports"
    "github.com/example/mmw-foosvc/internal/domain"
)

type GetFooHandler struct {
    repo ports.FooRepository
}

func NewGetFooHandler(repo ports.FooRepository) *GetFooHandler {
    return &GetFooHandler{repo: repo}
}

func (h *GetFooHandler) Handle(ctx context.Context, q dto.GetFooQuery) (*dto.FooDTO, error) {
    id, err := domain.FooIDFromString(q.ID)
    if err != nil {
        return nil, fmt.Errorf("get foo: %w", err)
    }

    foo, err := h.repo.FindByID(ctx, id)
    if err != nil {
        return nil, fmt.Errorf("get foo: %w", err)
    }

    return dto.FooDTOFromDomain(foo), nil
}
```

## DTOs

DTOs are plain structs with no domain logic. They cross the boundary
between the inbound adapter and the application layer.

```go
// modules/foosvc/internal/application/dto/foo_dto.go
package dto

import (
    "github.com/google/uuid"
    "github.com/example/mmw-foosvc/internal/domain"
)

type CreateFooCommand struct {
    Title    string
    Priority string
    OwnerID  uuid.UUID
}

type CompleteFooCommand struct {
    ID      string
    OwnerID uuid.UUID
}

type GetFooQuery struct {
    ID      string
    OwnerID uuid.UUID
}

type ListFoosQuery struct {
    OwnerID uuid.UUID
}

type FooDTO struct {
    ID       string
    Title    string
    Status   string
    Priority string
    OwnerID  string
}

func FooDTOFromDomain(f *domain.Foo) *FooDTO {
    return &FooDTO{
        ID:       f.ID().String(),
        Title:    f.Title().String(),
        Status:   string(f.Status()),
        Priority: string(f.Priority()),
        OwnerID:  f.OwnerID().String(),
    }
}
```
