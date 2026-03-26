# Contract Files Quick Reference

Minimal file content for a contract definition module. Copy and adjust
`foo` → your module name.

## File layout

```
contracts/definitions/foomod/
├── go.mod            (zero deps)
├── api.go            (interface)
├── dto.go            (request/response types)
├── errors.go         (public error sentinels)
└── inproc_client.go  (wraps any impl behind the interface)
```

## go.mod

```
module github.com/example/mmw-contracts/definitions/foomod

go 1.23
```

## api.go skeleton

```go
package foomod

import "context"

type FooService interface {
    // Add one method per use case exposed to other modules.
}
```

## inproc_client.go skeleton

```go
package foomod

import "context"

type InprocClient struct{ impl FooService }

func NewInprocClient(impl FooService) *InprocClient {
    return &InprocClient{impl: impl}
}

// Delegate every method to impl.
// Example:
// func (c *InprocClient) DoThing(ctx context.Context, req DoThingRequest) error {
//     return c.impl.DoThing(ctx, req)
// }
```

## Rule: The concrete *Module must satisfy FooService

In `modules/foomod/foomod.go`, implement each interface method as a public
method on `*Module`. In `cmd/mmw/main.go`, add a compile-time check:

```go
// This line will fail to compile if *foomod.Module doesn't satisfy FooService.
var _ deffoomod.FooService = (*foomod.Module)(nil)
```
