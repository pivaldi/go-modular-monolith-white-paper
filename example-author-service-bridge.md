
`bridge/authorsvc/go.mod`:

```go
module github.com/example/service-manager/bridge/authorsvc

go 1.22

// No dependencies - pure interfaces and DTOs
```

`bridge/authorsvc/api.go`:

```go
package authorsvc

import (
    "context"
    "time"
)

// AuthorService is the public API for the author service.
// Any service can import and use this interface.
//
// Implementations:
//   - InprocServer (wraps authorsvc/internal, in-process)
//   - Connect handler (network transport over HTTP)
type AuthorService interface {
    GetAuthor(ctx context.Context, id string) (*AuthorDTO, error)
    ListAuthors(ctx context.Context, req ListAuthorsRequest) (*ListAuthorsResponse, error)
    CreateAuthor(ctx context.Context, req CreateAuthorRequest) (*AuthorDTO, error)
    UpdateAuthor(ctx context.Context, req UpdateAuthorRequest) (*AuthorDTO, error)
}
```

**File: `bridge/authorsvc/dto.go`**

```go
package authorsvc

import "time"

// AuthorDTO is the public representation of an author.
// This decouples the bridge from internal domain models.
type AuthorDTO struct {
    ID        string
    Name      string
    Bio:       string
    Website:   string
    CreatedAt: time.Time,
    UpdatedAt: time.Time,
}

// Request/Response types
type ListAuthorsRequest struct {
    PageSize  int
    PageToken string
}

type ListAuthorsResponse struct {
    Authors       []*AuthorDTO
    NextPageToken string
}

type CreateAuthorRequest struct {
    Name    string
    Bio:       string
    Website:   string
}

type UpdateAuthorRequest struct {
    ID      string
    Name    string
    Bio:       string
    Website:   string
    CreatedAt: time.Time,
    UpdatedAt: time.Time,
}
```

`bridge/authorsvc/errors.go`:

```go
package authorsvc

import "errors"

// Public errors that callers can check
var (
    ErrAuthorNotFound   = errors.New("author not found")
    ErrInvalidAuthorName = errors.New("invalid author name")
    ErrDuplicateAuthor  = errors.New("author already exists")
)
```
