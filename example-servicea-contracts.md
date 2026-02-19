# Service A Contract Definition

The contract definition contains ONLY the public API contract: interfaces, DTOs, errors, and the thin client wrapper. It has **zero dependencies**.

## Contract Definition Structure

```
contracts/definitions/serviceasvc/
├── go.mod              # No dependencies (literally zero!)
├── api.go              # ServiceAService interface
├── dto.go              # Data transfer objects
├── errors.go           # Public error types
└── inproc_client.go    # Thin client wrapper
```

**Note:** InprocServer is NOT in the contract definition. It lives in `services/serviceasvc/internal/adapters/inbound/contracts/inproc_server.go` as an inbound adapter.

---

## Contract Definition Files

### go.mod

`contracts/definitions/serviceasvc/go.mod`:

```go
module github.com/example/service-manager/contracts/definitions/serviceasvc

go 1.22

// No dependencies - pure interfaces and DTOs
```

`contracts/definitions/serviceasvc/api.go`:

```go
package serviceasvc

import (
    "context"
    "time"
)

// ServiceAService is the public API for Service A.
// Any service can import and use this interface.
//
// Implementations (live in service, NOT in contract definition):
//   - InprocServer in services/serviceasvc/internal/adapters/inbound/contracts/
//   - Connect handler in services/serviceasvc/internal/adapters/inbound/connect/
type ServiceAService interface {
    GetServiceA(ctx context.Context, id string) (*ServiceADTO, error)
    ListServiceAs(ctx context.Context, req ListServiceAsRequest) (*ListServiceAsResponse, error)
    CreateServiceA(ctx context.Context, req CreateServiceARequest) (*ServiceADTO, error)
    UpdateServiceA(ctx context.Context, req UpdateServiceARequest) (*ServiceADTO, error)
}
```

**File: `contracts/definitions/serviceasvc/dto.go`**

```go
package serviceasvc

import "time"

// ServiceADTO is the public representation of a Service A entity.
// This decouples the contract definition from internal domain models.
type ServiceADTO struct {
    ID        string
    Name      string
    Bio       string
    Website   string
    CreatedAt time.Time
    UpdatedAt time.Time
}

// Request/Response types
type ListServiceAsRequest struct {
    PageSize  int
    PageToken string
}

type ListServiceAsResponse struct {
    ServiceAs     []*ServiceADTO
    NextPageToken string
}

type CreateServiceARequest struct {
    Name    string
    Bio     string
    Website string
}

type UpdateServiceARequest struct {
    ID        string
    Name      string
    Bio       string
    Website   string
    CreatedAt time.Time
    UpdatedAt time.Time
}
```

`contracts/definitions/serviceasvc/errors.go`:

```go
package serviceasvc

import "errors"

// Public errors that callers can check
var (
    ErrServiceANotFound   = errors.New("service a entity not found")
    ErrInvalidServiceAName = errors.New("invalid service a name")
    ErrDuplicateServiceA  = errors.New("service a already exists")
)
```
