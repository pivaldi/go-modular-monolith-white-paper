# Complete Contract Implementation: Service A

This document shows the complete contract-based architecture with clear separation between the public contract (contract definition) and the implementation (service adapter).

## Part 1: Contract Definition (Public Contract)

**Location:** `contracts/definitions/serviceasvc/`

The contract definition contains ONLY the public API: interfaces, DTOs, errors, and the thin client wrapper. It has **zero dependencies** - no `require` statements in go.mod, no imports of `internal/` packages.

### inproc_client.go

```go
// contracts/definitions/serviceasvc/inproc_client.go
package serviceasvc

import "context"

// InprocClient implements ServiceAService by calling any ServiceAService implementation.
// It depends on the INTERFACE, not a concrete implementation.
//
// Performance: Direct function call - <1μs latency, zero serialization overhead.
type InprocClient struct {
    server ServiceAService  // Interface reference (not *InprocServer!)
}

// NewInprocClient creates a new in-process client.
// Accepts any implementation of the ServiceAService interface.
func NewInprocClient(server ServiceAService) *InprocClient {
    return &InprocClient{server: server}
}

func (c *InprocClient) GetServiceA(ctx context.Context, id string) (*ServiceADTO, error) {
    // Direct function call via interface - no network, no serialization
    return c.server.GetServiceA(ctx, id)
}

func (c *InprocClient) CreateServiceA(ctx context.Context, req CreateServiceARequest) (*ServiceADTO, error) {
    return c.server.CreateServiceA(ctx, req)
}

func (c *InprocClient) ListServiceAs(ctx context.Context, req ListServiceAsRequest) (*ListServiceAsResponse, error) {
    return c.server.ListServiceAs(ctx, req)
}

func (c *InprocClient) UpdateServiceA(ctx context.Context, req UpdateServiceARequest) (*ServiceADTO, error) {
    return c.server.UpdateServiceA(ctx, req)
}
```

**Key Points:**
- InprocClient holds `ServiceAService` interface, not `*InprocServer` concrete type
- Accepts ANY implementation of the interface
- Enables loose coupling and testability
- Consumer never knows about the concrete implementation

---

## Part 2: Service Adapter (Implementation)

**Location:** `services/serviceasvc/internal/adapters/inbound/contracts/`

InprocServer is an **inbound adapter** that implements the contract interface. It lives in the service's internal adapters because:
- It wraps the service's internal application layer
- It translates between contract DTOs and internal types
- It maps domain errors to contract errors
- Like all adapters, it belongs to the service that implements it

### inproc_server.go

```go
// services/serviceasvc/internal/adapters/inbound/contracts/inproc_server.go
package contracts

import (
    "context"
    "errors"

    // Import contract interface (public API)
    "github.com/example/service-manager/contracts/definitions/serviceasvc"

    // CAN import service internals - same Go module!
    "github.com/example/service-manager/services/serviceasvc/internal/application/command"
    "github.com/example/service-manager/services/serviceasvc/internal/application/query"
    "github.com/example/service-manager/services/serviceasvc/internal/domain/servicea"
)

// InprocServer is an inbound adapter that implements the serviceasvc.ServiceAService interface.
// It wraps the service's internal application layer and exposes it via the contract API.
type InprocServer struct {
    getServiceAQuery   *query.GetServiceAQuery
    listServiceAsQuery *query.ListServiceAsQuery
    createServiceACmd  *command.CreateServiceACommand
    updateServiceACmd  *command.UpdateServiceACommand
}

// NewInprocServer creates an InprocServer and returns it as the contract interface type.
// This enables loose coupling - callers depend on the interface, not the concrete type.
func NewInprocServer(
    getServiceAQuery *query.GetServiceAQuery,
    listServiceAsQuery *query.ListServiceAsQuery,
    createServiceACmd *command.CreateServiceACommand,
    updateServiceACmd *command.UpdateServiceACommand,
) serviceasvc.ServiceAService {
    return &InprocServer{
        getServiceAQuery:   getServiceAQuery,
        listServiceAsQuery: listServiceAsQuery,
        createServiceACmd:  createServiceACmd,
        updateServiceACmd:  updateServiceACmd,
    }
}

func (s *InprocServer) GetServiceA(ctx context.Context, id string) (*serviceasvc.ServiceADTO, error) {
    // Call internal application layer directly
    result, err := s.getServiceAQuery.Execute(ctx, id)
    if err != nil {
        // Translate domain errors to contract errors
        if errors.Is(err, servicea.ErrServiceANotFound) {
            return nil, serviceasvc.ErrServiceANotFound
        }
        return nil, err
    }

    // Map internal DTO to contract DTO
    return &serviceasvc.ServiceADTO{
        ID:        result.ID,
        Name:      result.Name,
        Bio:       result.Bio,
        Website:   result.Website,
        AvatarURL: result.AvatarURL,
        CreatedAt: result.CreatedAt,
        UpdatedAt: result.UpdatedAt,
    }, nil
}

func (s *InprocServer) CreateServiceA(ctx context.Context, req serviceasvc.CreateServiceARequest) (*serviceasvc.ServiceADTO, error) {
    input := command.CreateServiceAInput{
        Name:    req.Name,
        Bio:     req.Bio,
        Website: req.Website,
    }

    result, err := s.createServiceACmd.Execute(ctx, input)
    if err != nil {
        // Translate domain errors to contract errors
        if errors.Is(err, servicea.ErrInvalidName) {
            return nil, serviceasvc.ErrInvalidServiceAName
        }
        if errors.Is(err, servicea.ErrDuplicateServiceA) {
            return nil, serviceasvc.ErrDuplicateServiceA
        }
        return nil, err
    }

    return &serviceasvc.ServiceADTO{
        ID:        result.ID,
        Name:      result.Name,
        Bio:       result.Bio,
        Website:   result.Website,
        CreatedAt: result.CreatedAt,
        UpdatedAt: result.UpdatedAt,
    }, nil
}

func (s *InprocServer) ListServiceAs(ctx context.Context, req serviceasvc.ListServiceAsRequest) (*serviceasvc.ListServiceAsResponse, error) {
    input := query.ListServiceAsInput{
        PageSize:  req.PageSize,
        PageToken: req.PageToken,
    }

    result, err := s.listServiceAsQuery.Execute(ctx, input)
    if err != nil {
        return nil, err
    }

    // Map results
    var serviceAs []*serviceasvc.ServiceADTO
    for _, a := range result.ServiceAs {
        serviceAs = append(serviceAs, &serviceasvc.ServiceADTO{
            ID:        a.ID,
            Name:      a.Name,
            Bio:       a.Bio,
            Website:   a.Website,
            AvatarURL: a.AvatarURL,
            CreatedAt: a.CreatedAt,
            UpdatedAt: a.UpdatedAt,
        })
    }

    return &serviceasvc.ListServiceAsResponse{
        ServiceAs:     serviceAs,
        NextPageToken: result.NextPageToken,
    }, nil
}

func (s *InprocServer) UpdateServiceA(ctx context.Context, req serviceasvc.UpdateServiceARequest) (*serviceasvc.ServiceADTO, error) {
    input := command.UpdateServiceAInput{
        ID:      req.ID,
        Name:    req.Name,
        Bio:     req.Bio,
        Website: req.Website,
    }

    result, err := s.updateServiceACmd.Execute(ctx, input)
    if err != nil {
        if errors.Is(err, servicea.ErrServiceANotFound) {
            return nil, serviceasvc.ErrServiceANotFound
        }
        return nil, err
    }

    return &serviceasvc.ServiceADTO{
        ID:        result.ID,
        Name:      result.Name,
        Bio:       result.Bio,
        Website:   result.Website,
        UpdatedAt: result.UpdatedAt,
    }, nil
}
```

---

## Key Architectural Insights

### 1. InprocServer is an Adapter
InprocServer lives in `services/serviceasvc/internal/adapters/inbound/contracts/`, not in the contract definition. This is intentional:
- It's an **inbound adapter** that implements the contract interface
- It wraps the service's internal application layer
- It belongs to the service, just like HTTP or Connect adapters

### 2. Contract Definition Remains Truly Dependency-Free
The contract definition (`contracts/definitions/serviceasvc/`) contains ONLY:
- Interfaces (api.go)
- DTOs (dto.go)
- Errors (errors.go)
- InprocClient (thin wrapper)

It has **zero dependencies** - literally no `require` statements in go.mod, no imports of `internal/` packages.

### 3. Interface-Based Coupling
InprocClient holds a `ServiceAService` interface reference, not `*InprocServer` concrete type. This enables:
- Loose coupling between consumer and provider
- Easy testing (mock the interface)
- Swappable implementations (in-process vs network)

### 4. Factory Returns Interface
`NewInprocServer()` returns `serviceasvc.ServiceAService` interface type, not `*InprocServer`. This ensures:
- Callers depend on the interface, not the concrete type
- Implementation details are hidden
- Easier to refactor implementation without breaking consumers

### 5. Clear Separation of Concerns
- **Contract Definition**: Public contract definition (what can be called)
- **Service Adapter**: Implementation details (how it's implemented)
- **Consumer**: Only knows about the contract interface

This is the essence of true module independence: contract definitions define contracts, services provide implementations.
