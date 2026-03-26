# Complete Protobuf + Connect Workflow

End-to-end: from `.proto` definition to a running Connect handler.

## Directory Layout

```
contracts/
├── buf.yaml                        ← buf workspace config
├── buf.gen.yaml                    ← code generation config
├── proto/
│   └── foo/v1/
│       └── foo.proto
└── gen/
    └── go/
        └── foo/v1/
            ├── foo.pb.go           ← protobuf messages
            └── foov1connect/
                └── foo.connect.go  ← Connect handler interface + client
```

## Step 1: Define the Proto

```protobuf
// contracts/proto/foo/v1/foo.proto
syntax = "proto3";

package foo.v1;

option go_package = "github.com/example/mmw-contracts/gen/go/foo/v1;foov1";

service FooService {
  rpc CreateFoo(CreateFooRequest) returns (CreateFooResponse);
  rpc GetFoo(GetFooRequest)       returns (GetFooResponse);
  rpc CompleteFoo(CompleteFooRequest) returns (CompleteFooResponse);
  rpc DeleteFoo(DeleteFooRequest) returns (DeleteFooResponse);
  rpc ListFoos(ListFoosRequest)   returns (ListFoosResponse);
}

message CreateFooRequest  { string title = 1; string priority = 2; }
message CreateFooResponse { Foo foo = 1; }
message GetFooRequest     { string id = 1; }
message GetFooResponse    { Foo foo = 1; }
message CompleteFooRequest { string id = 1; }
message CompleteFooResponse {}
message DeleteFooRequest  { string id = 1; }
message DeleteFooResponse {}
message ListFoosRequest   {}
message ListFoosResponse  { repeated Foo foos = 1; }

message Foo {
  string id       = 1;
  string title    = 2;
  string status   = 3;
  string priority = 4;
  string owner_id = 5;
}
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
# contracts/buf.gen.yaml
version: v2
plugins:
  - remote: buf.build/protocolbuffers/go
    out: gen/go
    opt:
      - paths=source_relative
  - remote: buf.build/connectrpc/go
    out: gen/go
    opt:
      - paths=source_relative
```

## Step 3: Generate Code

```bash
cd contracts && buf generate
```

This produces:
- `gen/go/foo/v1/foo.pb.go` — Go structs for all messages
- `gen/go/foo/v1/foov1connect/foo.connect.go` — `FooServiceHandler` interface + `NewFooServiceHandler` + `NewFooServiceClient`

## Step 4: Implement the Handler

```go
// modules/foosvc/internal/adapters/inbound/connect/foo_handler.go
package connectadapter

import (
    "context"
    "connectrpc.com/connect"
    foov1 "github.com/example/mmw-contracts/gen/go/foo/v1"
    "github.com/example/mmw-contracts/gen/go/foo/v1/foov1connect"
    "github.com/example/mmw-foosvc/internal/application"
    "github.com/example/mmw-foosvc/internal/application/dto"
)

type FooHandler struct{ svc *application.FooApplicationService }

var _ foov1connect.FooServiceHandler = (*FooHandler)(nil)

func NewFooHandler(svc *application.FooApplicationService) *FooHandler {
    return &FooHandler{svc: svc}
}

func (h *FooHandler) CreateFoo(
    ctx context.Context,
    req *connect.Request[foov1.CreateFooRequest],
) (*connect.Response[foov1.CreateFooResponse], error) {
    ownerID, _ := ownerIDFromContext(ctx)
    result, err := h.svc.CreateFoo(ctx, dto.CreateFooCommand{
        Title:    req.Msg.Title,
        Priority: req.Msg.Priority,
        OwnerID:  ownerID,
    })
    if err != nil {
        return nil, domainErrToConnect(err)
    }
    return connect.NewResponse(&foov1.CreateFooResponse{
        Foo: &foov1.Foo{
            Id:       result.ID,
            Title:    result.Title,
            Status:   result.Status,
            Priority: result.Priority,
            OwnerId:  result.OwnerID,
        },
    }), nil
}
```

## Step 5: Register in Module Factory

```go
// modules/foosvc/foosvc.go — inside New()
mux := http.NewServeMux()
path, handler := foov1connect.NewFooServiceHandler(
    connectadapter.NewFooHandler(appSvc),
    connect.WithInterceptors(oglconnect.NewErrorLoggingInterceptor(infra.Logger)),
)
mux.Handle(path, connectadapter.NewAuthMiddleware(
    infra.BarSvc, infra.Logger, nil, handler,
))
```

## Network vs In-Process

The same `FooHandler` is used for both. Only the composition root changes:

**In-process (monolith):**
```go
// cmd/mmw/main.go — no Connect client needed
barModule, _ := barsvc.New(barsvc.Infrastructure{
    FooSvc: deffoosvc.NewInprocClient(fooModule),
})
```

**Network (microservice split):**
```go
// cmd/bar/main.go — after extracting foosvc to its own process
fooClient := foov1connect.NewFooServiceClient(
    http.DefaultClient,
    "https://foosvc.internal",
)
barModule, _ := barsvc.New(barsvc.Infrastructure{
    FooSvc: fooClient, // satisfies deffoosvc.FooService if DTOs match
})
```
