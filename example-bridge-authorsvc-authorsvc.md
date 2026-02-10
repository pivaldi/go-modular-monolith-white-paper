# Complete Bridge Implementation: Author Service

This document shows the complete bridge pattern with clear separation between the public contract (bridge module) and the implementation (service adapter).

## Part 1: Bridge Module (Public Contract)

**Location:** `bridge/authorsvc/`

The bridge module contains ONLY the public API: interfaces, DTOs, errors, and the thin client wrapper. It has **zero dependencies** - no `require` statements in go.mod, no imports of `internal/` packages.

### inproc_client.go

```go
// bridge/authorsvc/inproc_client.go
package authorsvc

import "context"

// InprocClient implements AuthorService by calling any AuthorService implementation.
// It depends on the INTERFACE, not a concrete implementation.
//
// Performance: Direct function call - <1μs latency, zero serialization overhead.
type InprocClient struct {
    server AuthorService  // Interface reference (not *InprocServer!)
}

// NewInprocClient creates a new in-process client.
// Accepts any implementation of the AuthorService interface.
func NewInprocClient(server AuthorService) *InprocClient {
    return &InprocClient{server: server}
}

func (c *InprocClient) GetAuthor(ctx context.Context, id string) (*AuthorDTO, error) {
    // Direct function call via interface - no network, no serialization
    return c.server.GetAuthor(ctx, id)
}

func (c *InprocClient) CreateAuthor(ctx context.Context, req CreateAuthorRequest) (*AuthorDTO, error) {
    return c.server.CreateAuthor(ctx, req)
}

func (c *InprocClient) ListAuthors(ctx context.Context, req ListAuthorsRequest) (*ListAuthorsResponse, error) {
    return c.server.ListAuthors(ctx, req)
}

func (c *InprocClient) UpdateAuthor(ctx context.Context, req UpdateAuthorRequest) (*AuthorDTO, error) {
    return c.server.UpdateAuthor(ctx, req)
}
```

**Key Points:**
- InprocClient holds `AuthorService` interface, not `*InprocServer` concrete type
- Accepts ANY implementation of the interface
- Enables loose coupling and testability
- Consumer never knows about the concrete implementation

---

## Part 2: Service Adapter (Implementation)

**Location:** `services/authorsvc/internal/adapters/inbound/bridge/`

InprocServer is an **inbound adapter** that implements the bridge interface. It lives in the service's internal adapters because:
- It wraps the service's internal application layer
- It translates between bridge DTOs and internal types
- It maps domain errors to bridge errors
- Like all adapters, it belongs to the service that implements it

### inproc_server.go

```go
// services/authorsvc/internal/adapters/inbound/bridge/inproc_server.go
package bridge

import (
    "context"
    "errors"

    // Import bridge interface (public API)
    "github.com/example/service-manager/bridge/authorsvc"

    // CAN import service internals - same Go module!
    "github.com/example/service-manager/services/authorsvc/internal/application/command"
    "github.com/example/service-manager/services/authorsvc/internal/application/query"
    "github.com/example/service-manager/services/authorsvc/internal/domain/author"
)

// InprocServer is an inbound adapter that implements the bridge.AuthorService interface.
// It wraps the service's internal application layer and exposes it via the bridge API.
type InprocServer struct {
    getAuthorQuery   *query.GetAuthorQuery
    listAuthorsQuery *query.ListAuthorsQuery
    createAuthorCmd  *command.CreateAuthorCommand
    updateAuthorCmd  *command.UpdateAuthorCommand
}

// NewInprocServer creates an InprocServer and returns it as the bridge interface type.
// This enables loose coupling - callers depend on the interface, not the concrete type.
func NewInprocServer(
    getAuthorQuery *query.GetAuthorQuery,
    listAuthorsQuery *query.ListAuthorsQuery,
    createAuthorCmd *command.CreateAuthorCommand,
    updateAuthorCmd *command.UpdateAuthorCommand,
) authorsvc.AuthorService {
    return &InprocServer{
        getAuthorQuery:   getAuthorQuery,
        listAuthorsQuery: listAuthorsQuery,
        createAuthorCmd:  createAuthorCmd,
        updateAuthorCmd:  updateAuthorCmd,
    }
}

func (s *InprocServer) GetAuthor(ctx context.Context, id string) (*authorsvc.AuthorDTO, error) {
    // Call internal application layer directly
    result, err := s.getAuthorQuery.Execute(ctx, id)
    if err != nil {
        // Translate domain errors to bridge errors
        if errors.Is(err, author.ErrAuthorNotFound) {
            return nil, authorsvc.ErrAuthorNotFound
        }
        return nil, err
    }

    // Map internal DTO to bridge DTO
    return &authorsvc.AuthorDTO{
        ID:        result.ID,
        Name:      result.Name,
        Bio:       result.Bio,
        Website:   result.Website,
        AvatarURL: result.AvatarURL,
        CreatedAt: result.CreatedAt,
        UpdatedAt: result.UpdatedAt,
    }, nil
}

func (s *InprocServer) CreateAuthor(ctx context.Context, req authorsvc.CreateAuthorRequest) (*authorsvc.AuthorDTO, error) {
    input := command.CreateAuthorInput{
        Name:    req.Name,
        Bio:     req.Bio,
        Website: req.Website,
    }

    result, err := s.createAuthorCmd.Execute(ctx, input)
    if err != nil {
        // Translate domain errors to bridge errors
        if errors.Is(err, author.ErrInvalidName) {
            return nil, authorsvc.ErrInvalidAuthorName
        }
        if errors.Is(err, author.ErrDuplicateAuthor) {
            return nil, authorsvc.ErrDuplicateAuthor
        }
        return nil, err
    }

    return &authorsvc.AuthorDTO{
        ID:        result.ID,
        Name:      result.Name,
        Bio:       result.Bio,
        Website:   result.Website,
        CreatedAt: result.CreatedAt,
        UpdatedAt: result.UpdatedAt,
    }, nil
}

func (s *InprocServer) ListAuthors(ctx context.Context, req authorsvc.ListAuthorsRequest) (*authorsvc.ListAuthorsResponse, error) {
    input := query.ListAuthorsInput{
        PageSize:  req.PageSize,
        PageToken: req.PageToken,
    }

    result, err := s.listAuthorsQuery.Execute(ctx, input)
    if err != nil {
        return nil, err
    }

    // Map results
    var authors []*authorsvc.AuthorDTO
    for _, a := range result.Authors {
        authors = append(authors, &authorsvc.AuthorDTO{
            ID:        a.ID,
            Name:      a.Name,
            Bio:       a.Bio,
            Website:   a.Website,
            AvatarURL: a.AvatarURL,
            CreatedAt: a.CreatedAt,
            UpdatedAt: a.UpdatedAt,
        })
    }

    return &authorsvc.ListAuthorsResponse{
        Authors:       authors,
        NextPageToken: result.NextPageToken,
    }, nil
}

func (s *InprocServer) UpdateAuthor(ctx context.Context, req authorsvc.UpdateAuthorRequest) (*authorsvc.AuthorDTO, error) {
    input := command.UpdateAuthorInput{
        ID:      req.ID,
        Name:    req.Name,
        Bio:     req.Bio,
        Website: req.Website,
    }

    result, err := s.updateAuthorCmd.Execute(ctx, input)
    if err != nil {
        if errors.Is(err, author.ErrAuthorNotFound) {
            return nil, authorsvc.ErrAuthorNotFound
        }
        return nil, err
    }

    return &authorsvc.AuthorDTO{
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
InprocServer lives in `services/authorsvc/internal/adapters/inbound/bridge/`, not in the bridge module. This is intentional:
- It's an **inbound adapter** that implements the bridge interface
- It wraps the service's internal application layer
- It belongs to the service, just like HTTP or Connect adapters

### 2. Bridge Remains Truly Dependency-Free
The bridge module (`bridge/authorsvc/`) contains ONLY:
- Interfaces (api.go)
- DTOs (dto.go)
- Errors (errors.go)
- InprocClient (thin wrapper)

It has **zero dependencies** - literally no `require` statements in go.mod, no imports of `internal/` packages.

### 3. Interface-Based Coupling
InprocClient holds an `AuthorService` interface reference, not `*InprocServer` concrete type. This enables:
- Loose coupling between consumer and provider
- Easy testing (mock the interface)
- Swappable implementations (in-process vs network)

### 4. Factory Returns Interface
`NewInprocServer()` returns `authorsvc.AuthorService` interface type, not `*InprocServer`. This ensures:
- Callers depend on the interface, not the concrete type
- Implementation details are hidden
- Easier to refactor implementation without breaking consumers

### 5. Clear Separation of Concerns
- **Bridge**: Public contract definition (what can be called)
- **Service Adapter**: Implementation details (how it's implemented)
- **Consumer**: Only knows about the bridge interface

This is the essence of true module independence: bridge modules define contracts, services provide implementations.
