# Application Layer Examples

The application layer orchestrates use cases. It has no business logic
(that lives in the domain) and no infrastructure knowledge (that lives
in adapters). It depends only on domain types and port interfaces.

## Ports (Secondary Interfaces)

Ports are interfaces the application layer defines. Adapters implement them.

```go
// modules/todo/internal/application/ports/repository.go
package ports

import (
    "context"
    "github.com/google/uuid"
    "github.com/pivaldi/mmw-todo/internal/domain"
)

type TodoRepository interface {
    Save(ctx context.Context, todo *domain.Todo) error
    FindByID(ctx context.Context, id domain.TodoID) (*domain.Todo, error)
    FindByUserID(ctx context.Context, userID uuid.UUID) ([]*domain.Todo, error)
    Delete(ctx context.Context, id domain.TodoID) error
    Health(ctx context.Context) (any, error)
}
```

```go
// modules/todo/internal/application/ports/events.go
type EventDispatcher interface {
    Dispatch(ctx context.Context, events []domain.DomainEvent) error
}
```

```go
// modules/todo/internal/application/ports/uow.go (via platform)
type UnitOfWork interface {
    WithTransaction(ctx context.Context, fn func(ctx context.Context) error) error
    Executor(ctx context.Context) db.DBExecutor
}
```

## Application Service (Interface + Facade)

The `TodoService` interface is what the inbound adapter depends on — this allows
the handler to be tested with a real service backed by mocked ports.

```go
// modules/todo/internal/application/service.go
package application

type TodoService interface {
    CreateTodo(ctx context.Context, req *todov1.CreateTodoRequest) (*todov1.CreateTodoResponse, error)
    GetTodo(ctx context.Context, req *todov1.GetTodoRequest) (*todov1.GetTodoResponse, error)
    UpdateTodo(ctx context.Context, req *todov1.UpdateTodoRequest) (*todov1.UpdateTodoResponse, error)
    CompleteTodo(ctx context.Context, req *todov1.CompleteTodoRequest) (*todov1.CompleteTodoResponse, error)
    ReopenTodo(ctx context.Context, req *todov1.ReopenTodoRequest) (*todov1.ReopenTodoResponse, error)
    DeleteTodo(ctx context.Context, req *todov1.DeleteTodoRequest) (*todov1.DeleteTodoResponse, error)
    ListTodos(ctx context.Context, req *todov1.ListTodosRequest) (*todov1.ListTodosResponse, error)
    Health(ctx context.Context) (any, error)
}

// TodoApplicationService is the concrete implementation of TodoService.
type TodoApplicationService struct {
    createTodo  *command.CreateTodoCommand
    updateTodo  *command.UpdateTodoCommand
    completeTodo *command.CompleteTodoCommand
    reopenTodo  *command.ReopenTodoCommand
    deleteTodo  *command.DeleteTodoCommand
    getTodo     *query.GetTodoQuery
    listTodos   *query.ListTodosQuery
    repo        ports.TodoRepository
}

func NewTodoApplicationService(
    repo ports.TodoRepository,
    uow ports.UnitOfWork,
    dispatcher ports.EventDispatcher,
) *TodoApplicationService {
    return &TodoApplicationService{
        createTodo:   command.NewCreateTodoCommand(repo, uow, dispatcher),
        updateTodo:   command.NewUpdateTodoCommand(repo, uow, dispatcher),
        completeTodo: command.NewCompleteTodoCommand(repo, uow, dispatcher),
        reopenTodo:   command.NewReopenTodoCommand(repo, uow, dispatcher),
        deleteTodo:   command.NewDeleteTodoCommand(repo, uow, dispatcher),
        getTodo:      query.NewGetTodoQuery(repo),
        listTodos:    query.NewListTodosQuery(repo),
        repo:         repo,
    }
}
```

## Command Handler (Write Side)

Command handlers perform writes. They use the Unit of Work to wrap the
operation in a transaction, then dispatch domain events to the outbox.

```go
// modules/todo/internal/application/command/create_todo.go
package command

type CreateTodoCommand struct {
    repository      ports.TodoRepository
    unitOfWork      ports.UnitOfWork
    eventDispatcher ports.EventDispatcher
}

func NewCreateTodoCommand(
    repo ports.TodoRepository,
    uow ports.UnitOfWork,
    dispatcher ports.EventDispatcher,
) *CreateTodoCommand {
    return &CreateTodoCommand{repository: repo, unitOfWork: uow, eventDispatcher: dispatcher}
}

func (c *CreateTodoCommand) Execute(
    ctx context.Context,
    req *todov1.CreateTodoRequest,
) (*todov1.CreateTodoResponse, error) {
    userID, ok := appctx.UserID(ctx) // extracted from JWT by auth middleware
    if !ok {
        return nil, platform.NewDomainError(platform.CodeUnauthenticated, "missing user id")
    }

    title, err := domain.NewTitle(req.GetTitle())
    if err != nil { return nil, err }

    var resp *todov1.CreateTodoResponse
    err = c.unitOfWork.WithTransaction(ctx, func(txCtx context.Context) error {
        todo, err := domain.NewTodo(title, desc, dueDate, userID)
        if err != nil { return err }

        if err := c.repository.Save(txCtx, todo); err != nil { return err }

        // Dispatch to outbox table — same transaction as the domain write
        if err := c.eventDispatcher.Dispatch(txCtx, todo.Events()); err != nil { return err }

        resp = mapToProtoResponse(todo)
        return nil
    })
    return resp, err
}
```

## Query Handler (Read Side)

Query handlers perform reads only — no writes, no events, no Unit of Work.

```go
// modules/todo/internal/application/query/get_todo.go
package query

type GetTodoQuery struct {
    repo ports.TodoRepository
}

func NewGetTodoQuery(repo ports.TodoRepository) *GetTodoQuery {
    return &GetTodoQuery{repo: repo}
}

func (q *GetTodoQuery) Execute(
    ctx context.Context,
    req *todov1.GetTodoRequest,
) (*todov1.GetTodoResponse, error) {
    id, err := domain.TodoIDFromString(req.GetId())
    if err != nil {
        return nil, platform.NewDomainError(platform.CodeInvalidArgument, "invalid todo id")
    }

    todo, err := q.repo.FindByID(ctx, id)
    if err != nil {
        return nil, err
    }

    return &todov1.GetTodoResponse{Todo: mapToProto(todo)}, nil
}
```

## Wiring in the Module Factory

```go
// modules/todo/todo.go — newApplicationService()
func newApplicationService(infra Infrastructure) application.TodoService {
    uow := pfuow.New(infra.DBPool)
    todoRepo := postgres.NewPostgresTodoRepository(uow)
    eventDispatcher := events.NewPostgresOutboxDispatcher(uow)

    return application.NewTodoApplicationService(todoRepo, uow, eventDispatcher)
}
```
