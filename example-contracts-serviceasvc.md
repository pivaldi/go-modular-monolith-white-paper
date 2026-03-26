# Complete Contract Definition: foosvc

A worked example of all four files in `contracts/definitions/foosvc/`.

## go.mod

```go
module github.com/example/mmw-contracts/definitions/foosvc

go 1.23
// No require block — zero dependencies by design.
```

## api.go

```go
package foosvc

import "context"

type FooService interface {
    CreateFoo(ctx context.Context, req CreateFooRequest) (*FooResponse, error)
    GetFoo(ctx context.Context, req GetFooRequest) (*FooResponse, error)
    CompleteFoo(ctx context.Context, req CompleteFooRequest) error
    DeleteFoo(ctx context.Context, req DeleteFooRequest) error
    ListFoos(ctx context.Context, req ListFoosRequest) ([]*FooResponse, error)
}
```

## dto.go

```go
package foosvc

type CreateFooRequest   struct{ Title, Priority, OwnerID string }
type GetFooRequest      struct{ ID, OwnerID string }
type CompleteFooRequest struct{ ID, OwnerID string }
type DeleteFooRequest   struct{ ID, OwnerID string }
type ListFoosRequest    struct{ OwnerID string }

type FooResponse struct {
    ID, Title, Status, Priority, OwnerID string
}
```

## errors.go

```go
package foosvc

import "errors"

var (
    ErrNotFound   = errors.New("foosvc: foo not found")
    ErrForbidden  = errors.New("foosvc: access denied")
    ErrBadRequest = errors.New("foosvc: invalid request")
)
```

## inproc_client.go

```go
package foosvc

import "context"

type InprocClient struct{ impl FooService }

func NewInprocClient(impl FooService) *InprocClient {
    return &InprocClient{impl: impl}
}

func (c *InprocClient) CreateFoo(ctx context.Context, req CreateFooRequest) (*FooResponse, error) {
    return c.impl.CreateFoo(ctx, req)
}
func (c *InprocClient) GetFoo(ctx context.Context, req GetFooRequest) (*FooResponse, error) {
    return c.impl.GetFoo(ctx, req)
}
func (c *InprocClient) CompleteFoo(ctx context.Context, req CompleteFooRequest) error {
    return c.impl.CompleteFoo(ctx, req)
}
func (c *InprocClient) DeleteFoo(ctx context.Context, req DeleteFooRequest) error {
    return c.impl.DeleteFoo(ctx, req)
}
func (c *InprocClient) ListFoos(ctx context.Context, req ListFoosRequest) ([]*FooResponse, error) {
    return c.impl.ListFoos(ctx, req)
}
```

## How barsvc Uses This Contract

In `modules/barsvc/go.mod`:

```go
module github.com/example/mmw-barsvc

go 1.23

require github.com/example/mmw-contracts/definitions/foosvc v0.0.0
```

In `modules/barsvc/internal/application/ports/foo.go`:

```go
package ports

import deffoosvc "github.com/example/mmw-contracts/definitions/foosvc"

// FooServicePort aliases the contract interface for use in the application layer.
type FooServicePort = deffoosvc.FooService
```

In `cmd/mmw/main.go`:

```go
fooModule, _ := foosvc.New(foosvc.Infrastructure{...})

barModule, _ := barsvc.New(barsvc.Infrastructure{
    FooSvc: deffoosvc.NewInprocClient(fooModule),
    ...
})
```
