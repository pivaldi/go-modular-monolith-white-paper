# Complete Protobuf + Connect Workflow

End-to-end: from `.proto` definition to a running Connect handler.

## Directory Layout

```
poc/contracts/
├── buf.yaml                            ← buf workspace config (linting + breaking rules)
├── buf.gen.todo.yaml                   ← code generation config for todo domain
├── buf.gen.auth.yaml                   ← code generation config for auth domain
├── buf.gen.yaml                        ← runs protoc-gen-go-contracts custom plugin
├── proto/
│   ├── buf.yaml
│   ├── options/v1/options.proto        ← custom options (topic annotations)
│   ├── common/v1/errors.proto
│   ├── auth/v1/auth.proto
│   └── todo/v1/todo.proto
├── go/
│   ├── network/
│   │   ├── todo/v1/
│   │   │   ├── todo.pb.go              ← message structs (generated)
│   │   │   └── todov1connect/
│   │   │       └── todo.connect.go    ← Connect handler interface + client (generated)
│   │   └── auth/v1/…
│   └── application/
│       ├── todo/
│       │   ├── todo_service_contract_gen.go ← TodoService interface + noop (generated)
│       │   ├── events_gen.go               ← topic constants + type aliases (generated)
│       │   └── errors_gen.go               ← error code constants (generated)
│       └── auth/…
└── cmd/
    └── protoc-gen-go-contracts/        ← custom buf plugin source
```

## Step 1: Define the Proto

```protobuf
// contracts/proto/todo/v1/todo.proto
syntax = "proto3";

package todo.v1;

import "google/protobuf/timestamp.proto";
import "options/v1/options.proto";

option go_package = "github.com/pivaldi/mmw-contracts/gen/go/todo/v1;todov1";

service TodoService {
  rpc CreateTodo(CreateTodoRequest)   returns (CreateTodoResponse);
  rpc GetTodo(GetTodoRequest)         returns (GetTodoResponse);
  rpc UpdateTodo(UpdateTodoRequest)   returns (UpdateTodoResponse);
  rpc CompleteTodo(CompleteTodoRequest) returns (CompleteTodoResponse);
  rpc ReopenTodo(ReopenTodoRequest)   returns (ReopenTodoResponse);
  rpc DeleteTodo(DeleteTodoRequest)   returns (DeleteTodoResponse);
  rpc ListTodos(ListTodosRequest)     returns (ListTodosResponse);
}

enum TaskStatus {
  TASK_STATUS_UNSPECIFIED = 0;
  TASK_STATUS_PENDING     = 1;
  TASK_STATUS_IN_PROGRESS = 2;
  TASK_STATUS_COMPLETED   = 3;
  TASK_STATUS_CANCELLED   = 4;
}

enum Priority {
  PRIORITY_UNSPECIFIED = 0;
  PRIORITY_LOW         = 1;
  PRIORITY_MEDIUM      = 2;
  PRIORITY_HIGH        = 3;
  PRIORITY_URGENT      = 4;
}

message Todo {
  string id          = 1;
  string title       = 2;
  string description = 3;
  TaskStatus status  = 4;
  Priority priority  = 5;
  google.protobuf.Timestamp due_date   = 6;
  google.protobuf.Timestamp created_at = 7;
  google.protobuf.Timestamp updated_at = 8;
}

message CreateTodoRequest {
  string title       = 1;
  string description = 2;
  Priority priority  = 3;
  google.protobuf.Timestamp due_date = 4;
}
message CreateTodoResponse { Todo todo = 1; }
// … GetTodo, UpdateTodo, CompleteTodo, ReopenTodo, DeleteTodo, ListTodos
```

## Step 2: Configure buf

```yaml
# contracts/buf.yaml
version: v2
modules:
  - path: proto
lint:
  use:
    - STANDARD
breaking:
  use:
    - FILE
```

```yaml
# contracts/buf.gen.todo.yaml — standard plugins (network types + Connect stubs)
version: v2
managed:
  enabled: true
  override:
    - file_option: go_package_prefix
      value: github.com/pivaldi/mmw-contracts/go/network
inputs:
  - directory: .
    paths:
      - proto/todo
plugins:
  - remote: buf.build/protocolbuffers/go
    out: go/network          # → contracts/go/network/
    opt: paths=source_relative
  - remote: buf.build/connectrpc/go
    out: go/network          # → contracts/go/network/
    opt: paths=source_relative
```

```yaml
# contracts/buf.gen.yaml — custom protoc-gen-go-contracts plugin
version: v2
plugins:
  - local: protoc-gen-go-contracts
    out: go/application      # → contracts/go/application/
    opt: paths=source_relative
```

## Step 3: Generate Code

```bash
cd poc/contracts

# 1. Build the custom plugin once (or after plugin code changes):
go build -o $(go env GOPATH)/bin/protoc-gen-go-contracts ./cmd/protoc-gen-go-contracts

# 2. Run standard plugins (network types + Connect stubs):
buf generate --template buf.gen.todo.yaml

# 3. Run custom plugin (application-layer contracts):
buf generate --template buf.gen.yaml
```

This produces:
- `go/network/todo/v1/todo.pb.go` — Go structs for all messages
- `go/network/todo/v1/todov1connect/todo.connect.go` — `TodoServiceHandler` interface + `NewTodoServiceHandler` + `NewTodoServiceClient`
- `go/application/todo/todo_service_contract_gen.go` — `TodoService` interface + `NoopTodoService`
- `go/application/todo/events_gen.go` — topic constants + type aliases
- `go/application/todo/errors_gen.go` — error code constants

## Step 4: Implement the Handler

```go
// modules/todo/internal/adapters/inbound/connect/todo_handler.go
package connect

import (
    "context"
    "connectrpc.com/connect"
    todov1 "github.com/pivaldi/mmw-contracts/go/network/todo/v1"
    "github.com/pivaldi/mmw-todo/internal/application"
    "github.com/pivaldi/mmw-todo/internal/application/dto"
)

type TodoHandler struct{ service application.TodoService }

// Compile-time assertion: implements the generated Connect interface
var _ todov1connect.TodoServiceHandler = (*TodoHandler)(nil)

func NewTodoHandler(service application.TodoService) *TodoHandler {
    return &TodoHandler{service: service}
}

func (h *TodoHandler) CreateTodo(
    ctx context.Context,
    req *connect.Request[todov1.CreateTodoRequest],
) (*connect.Response[todov1.CreateTodoResponse], error) {
    appReq := dto.CreateTodoRequest{
        Title:       req.Msg.GetTitle(),
        Description: req.Msg.GetDescription(),
        Priority:    mapPriorityFromProto(req.Msg.GetPriority()),
    }
    if req.Msg.GetDueDate() != nil {
        dueDate := req.Msg.GetDueDate().AsTime()
        appReq.DueDate = &dueDate
    }
    todo, err := h.service.CreateTodo(ctx, &appReq)
    if err != nil {
        return nil, connectErrorFrom(err)
    }
    return connect.NewResponse(&todov1.CreateTodoResponse{
        Todo: mapTodoToProto(todo),
    }), nil
}
```

## Step 5: Register in Module Factory

```go
// modules/todo/todo.go — newHTTPServer()
mux := http.NewServeMux()
path, handler := todov1connect.NewTodoServiceHandler(
    connectadapter.NewTodoHandler(appSvc),
    connect.WithInterceptors(pfconnect.NewErrorLoggingInterceptor(infra.Logger)),
)
mux.Handle(path, pfmiddleware.NewAuth(tokenValidator, infra.Logger)(handler))
```

## Network vs In-Process

The same `TodoHandler` is used for both. Only the composition root changes:

**In-process (monolith):**
```go
// cmd/mmw/main.go
authModule, _ := auth.New(auth.Infrastructure{…})
todoModule, _ := todo.New(todo.Infrastructure{
    AuthSvc: authModule.PrivateService(), // direct in-process call
})
```

**Network (after extracting auth to its own pod):**
```go
// cmd/mmw/main.go — auth is now a separate service
todoModule, _ := todo.New(todo.Infrastructure{
    AuthSvc: defauth.NewPrivateHTTPClient(
        authv1connect.NewAuthPrivateServiceClient(&http.Client{}, "https://auth.internal"),
    ),
})
```

The `defauth.AuthPrivateService` interface is the same in both cases.
The todo module's application code never changes.
