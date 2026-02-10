`bridge/authorsvc/inproc_server.go`:

```go
package authorsvc

import (
	"context"
	"errors"

	// CAN import authorsvc internals - they're part of the same logical service
	"github.com/example/service-manager/services/authorsvc/internal/application/command"
	"github.com/example/service-manager/services/authorsvc/internal/application/query"
	"github.com/example/service-manager/services/authorsvc/internal/domain/author"
)

// InprocServer implements AuthorService by calling authorsvc internals directly.
// This is the "server side" of the in-process bridge.
type InprocServer struct {
	getAuthorQuery   *query.GetAuthorQuery
	listAuthorsQuery *query.ListAuthorsQuery
	createAuthorCmd  *command.CreateAuthorCommand
	updateAuthorCmd  *command.UpdateAuthorCommand
}

// NewInprocServer creates a new in-process server.
// Called from authorsvc/cmd/main.go during service startup.
func NewInprocServer(
	getAuthorQuery *query.GetAuthorQuery,
	listAuthorsQuery *query.ListAuthorsQuery,
	createAuthorCmd *command.CreateAuthorCommand,
	updateAuthorCmd *command.UpdateAuthorCommand,
) *InprocServer {
	return &InprocServer{
		getAuthorQuery:   getAuthorQuery,
		listAuthorsQuery: listAuthorsQuery,
		createAuthorCmd:  createAuthorCmd,
		updateAuthorCmd:  updateAuthorCmd,
	}
}

func (s *InprocServer) GetAuthor(ctx context.Context, id string) (*AuthorDTO, error) {
	// Call internal application layer directly
	result, err := s.getAuthorQuery.Execute(ctx, id)
	if err != nil {
		// Translate domain errors to bridge errors
		if errors.Is(err, author.ErrAuthorNotFound) {
			return nil, ErrAuthorNotFound
		}
		return nil, err
	}

	// Map internal DTO to bridge DTO
	return &AuthorDTO{
		ID:        result.ID,
		Name:      result.Name,
		Bio:       result.Bio,
		Website:   result.Website,
		AvatarURL: result.AvatarURL,
		CreatedAt: result.CreatedAt,
		UpdatedAt: result.UpdatedAt,
	}, nil
}

func (s *InprocServer) CreateAuthor(ctx context.Context, req CreateAuthorRequest) (*AuthorDTO, error) {
	input := command.CreateAuthorInput{
		Name:    req.Name,
		Bio:     req.Bio,
		Website: req.Website,
	}

	result, err := s.createAuthorCmd.Execute(ctx, input)
	if err != nil {
		if errors.Is(err, author.ErrInvalidName) {
			return nil, ErrInvalidAuthorName
		}
		if errors.Is(err, author.ErrDuplicateAuthor) {
			return nil, ErrDuplicateAuthor
		}
		return nil, err
	}

	return &AuthorDTO{
		ID:        result.ID,
		Name:      result.Name,
		Bio:       result.Bio,
		Website:   result.Website,
		CreatedAt: result.CreatedAt,
		UpdatedAt: result.UpdatedAt,
	}, nil
}

func (s *InprocServer) ListAuthors(ctx context.Context, req ListAuthorsRequest) (*ListAuthorsResponse, error) {
	input := query.ListAuthorsInput{
		PageSize:  req.PageSize,
		PageToken: req.PageToken,
	}

	result, err := s.listAuthorsQuery.Execute(ctx, input)
	if err != nil {
		return nil, err
	}

	// Map results
	var authors []*AuthorDTO
	for _, a := range result.Authors {
		authors = append(authors, &AuthorDTO{
			ID:        a.ID,
			Name:      a.Name,
			Bio:       a.Bio,
			Website:   a.Website,
			AvatarURL: a.AvatarURL,
			CreatedAt: a.CreatedAt,
			UpdatedAt: a.UpdatedAt,
		})
	}

	return &ListAuthorsResponse{
		Authors:       authors,
		NextPageToken: result.NextPageToken,
	}, nil
}

func (s *InprocServer) UpdateAuthor(ctx context.Context, req UpdateAuthorRequest) (*AuthorDTO, error) {
	input := command.UpdateAuthorInput{
		ID:      req.ID,
		Name:    req.Name,
		Bio:     req.Bio,
		Website: req.Website,
	}

	result, err := s.updateAuthorCmd.Execute(ctx, input)
	if err != nil {
		if errors.Is(err, author.ErrAuthorNotFound) {
			return nil, ErrAuthorNotFound
		}
		return nil, err
	}

	return &AuthorDTO{
		ID:        result.ID,
		Name:      result.Name,
		Bio:       result.Bio,
		Website:   result.Website,
		UpdatedAt: result.UpdatedAt,
	}, nil
}
```

`bridge/authorsvc/inproc_client.go`:

```go
package authorsvc

import "context"

// InprocClient implements AuthorService by calling InprocServer directly.
// This is the "client side" of the in-process bridge.
//
// Performance: Direct function call - <1μs latency, zero serialization overhead.
type InprocClient struct {
    server *InprocServer
}

// NewInprocClient creates a new in-process client.
// Pass the same InprocServer instance that the authorsvc is using.
func NewInprocClient(server *InprocServer) *InprocClient {
    return &InprocClient{server: server}
}

func (c *InprocClient) GetAuthor(ctx context.Context, id string) (*AuthorDTO, error) {
    // Direct function call - no network, no serialization
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
