# Complete Contract Definition: foomod

A worked example of all four files in `contracts/definitions/foomod/`.

## go.mod

```go
module github.com/example/mmw-contracts/definitions/foomod

go 1.23
// No require block — zero dependencies by design.
```

## api.go

```go
package foomod

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
package foomod

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
package foomod

import "errors"

var (
    ErrNotFound   = errors.New("foomod: foo not found")
    ErrForbidden  = errors.New("foomod: access denied")
    ErrBadRequest = errors.New("foomod: invalid request")
)
```

## inproc_client.go

```go
package foomod

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

## How barmod Uses This Contract

In `modules/barmod/go.mod`:

```go
module github.com/example/mmw-barmod

go 1.23

require github.com/example/mmw-contracts/definitions/foomod v0.0.0
```

In `modules/barmod/internal/application/ports/foo.go`:

```go
package ports

import deffoomod "github.com/example/mmw-contracts/definitions/foomod"

// FooServicePort aliases the contract interface for use in the application layer.
type FooServicePort = deffoomod.FooService
```

In `cmd/mmw/main.go`:

```go
fooModule, _ := foomod.New(foomod.Infrastructure{...})

barModule, _ := barmod.New(barmod.Infrastructure{
    FooSvc: deffoomod.NewInprocClient(fooModule),
    ...
})
```
