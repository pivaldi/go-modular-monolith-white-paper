# Contract Definition Pattern

A contract definition is a tiny, zero-dependency Go module that defines
the public interface of one module. Any other module that needs to call
`foomod` imports only this contract — never `modules/foomod/internal/`.

## The Four Files

Every contract definition consists of exactly four files:

```
contracts/definitions/foomod/
├── go.mod           ← zero dependencies
├── api.go           ← the service interface
├── dto.go           ← request/response types
├── errors.go        ← public error sentinels
└── inproc_client.go ← wraps any FooService impl behind the interface
```

### `go.mod` — Zero Dependencies

```go
module github.com/example/mmw-contracts/definitions/foomod

go 1.23
// NO require statements. Ever.
```

This zero-dependency rule is enforced by the Go compiler. If a
dependency creeps in, `go build` will fail in modules that import this
contract because they don't have that dependency.

### `api.go` — The Interface

```go
// contracts/definitions/foomod/api.go
package foomod

import "context"

// FooService is the public contract for the foomod module.
// All callers depend on this interface, never on the concrete implementation.
type FooService interface {
    CreateFoo(ctx context.Context, req CreateFooRequest) (*FooResponse, error)
    GetFoo(ctx context.Context, req GetFooRequest) (*FooResponse, error)
    CompleteFoo(ctx context.Context, req CompleteFooRequest) error
    DeleteFoo(ctx context.Context, req DeleteFooRequest) error
    ListFoos(ctx context.Context, req ListFoosRequest) ([]*FooResponse, error)
}
```

### `dto.go` — Request/Response Types

```go
// contracts/definitions/foomod/dto.go
package foomod

type CreateFooRequest struct {
    Title    string
    Priority string
    OwnerID  string
}

type GetFooRequest struct {
    ID      string
    OwnerID string
}

type CompleteFooRequest struct {
    ID      string
    OwnerID string
}

type DeleteFooRequest struct {
    ID      string
    OwnerID string
}

type ListFoosRequest struct {
    OwnerID string
}

type FooResponse struct {
    ID       string
    Title    string
    Status   string
    Priority string
    OwnerID  string
}
```

### `errors.go` — Public Error Sentinels

```go
// contracts/definitions/foomod/errors.go
package foomod

import "errors"

var (
    ErrNotFound   = errors.New("foomod: foo not found")
    ErrForbidden  = errors.New("foomod: access denied")
    ErrBadRequest = errors.New("foomod: invalid request")
)
```

### `inproc_client.go` — In-Process Wrapper

The `InprocClient` wraps any implementation of the `FooService` contract
interface behind a type-safe wrapper. The composition root uses this to
inject `foomod` into modules that depend on it:

```go
// contracts/definitions/foomod/inproc_client.go
package foomod

import "context"

// InprocClient wraps any FooService implementation for in-process injection.
type InprocClient struct {
    impl FooService
}

// NewInprocClient wraps any FooService implementation.
// The composition root calls: deffoomod.NewInprocClient(fooModule)
// where *fooModule satisfies FooService.
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

The concrete `*foomod.Module` satisfies `FooService` directly — it
implements each method in its public surface. `NewInprocClient` accepts
the `FooService` interface, so any implementation (including test
doubles) can be injected.

## Dependency Graph

```
contracts/definitions/foomod/   ← zero deps
        ↑
modules/barmod/                 ← imports deffoomod, calls via InprocClient
        ↑
cmd/mmw/main.go                 ← creates fooModule, wraps in InprocClient,
                                   injects into barModule.Infrastructure
```

`modules/barmod` never imports `modules/foomod` — only the contract.
This is enforced by Go's module system: `barmod/go.mod` lists
`mmw-contracts/definitions/foomod` as a dependency, not `mmw-foomod`.

## What Prevents Cycles

Each contract definition has zero dependencies. A service can depend on
a contract, but a contract can never depend on a service. This makes
cycles structurally impossible.

```
❌ IMPOSSIBLE (compiler error):
contracts/definitions/foomod → modules/foomod (would be circular)

✅ VALID:
modules/barmod → contracts/definitions/foomod → (nothing)
```
