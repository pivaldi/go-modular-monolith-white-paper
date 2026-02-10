# Complete Protobuf Workflow

When you're ready to add network transport, follow this workflow.

## 1. Define Service Contract

**File: `contracts/author/v1/author.proto`**

```protobuf
syntax = "proto3";

package author.v1;

option go_package = "github.com/example/service-manager/contracts/author/v1;authorv1";

service AuthorService {
  rpc GetAuthor(GetAuthorRequest) returns (GetAuthorResponse) {}
  rpc CreateAuthor(CreateAuthorRequest) returns (CreateAuthorResponse) {}
}

message Author {
  string id = 1;
  string name = 2;
  string bio = 3;
  string website = 4;
  int64 created_at = 5;
  int64 updated_at = 6;
}

message GetAuthorRequest {
  string id = 1;
}

message GetAuthorResponse {
  Author author = 1;
}

message CreateAuthorRequest {
  string name = 1;
  string bio = 2;
  string website = 3;
}

message CreateAuthorResponse {
  Author author = 1;
}
```

## 2. Generate Code

```bash
# Using buf
buf generate

# Generated files:
# - contracts/author/v1/author.pb.go (protobuf types)
# - contracts/author/v1/authorconnect/author.connect.go (Connect stubs)
```

## 3. Implement Connect Handler (Inbound Adapter)

```go
// services/authorsvc/internal/adapters/inbound/connect/handlers/author_handler.go
package handlers

import (
    "context"
    "connectrpc.com/connect"

    authorv1 "github.com/example/service-manager/contracts/author/v1"
    "github.com/example/service-manager/contracts/author/v1/authorconnect"
    "github.com/example/service-manager/services/authorsvc/internal/application/command"
    "github.com/example/service-manager/services/authorsvc/internal/application/query"
)

type AuthorHandler struct {
    getAuthorQuery  *query.GetAuthorQuery
    createAuthorCmd *command.CreateAuthorCommand
}

func NewAuthorHandler(
    getAuthorQuery *query.GetAuthorQuery,
    createAuthorCmd *command.CreateAuthorCommand,
) *AuthorHandler {
    return &AuthorHandler{
        getAuthorQuery:  getAuthorQuery,
        createAuthorCmd: createAuthorCmd,
    }
}

// Ensure we implement the interface
var _ authorconnect.AuthorServiceHandler = (*AuthorHandler)(nil)

func (h *AuthorHandler) GetAuthor(
    ctx context.Context,
    req *connect.Request[authorv1.GetAuthorRequest],
) (*connect.Response[authorv1.GetAuthorResponse], error) {
    // Call application layer
    result, err := h.getAuthorQuery.Execute(ctx, req.Msg.Id)
    if err != nil {
        return nil, connect.NewError(connect.CodeNotFound, err)
    }

    // Map to protobuf
    author := &authorv1.Author{
        Id:        result.ID,
        Name:      result.Name,
        Bio:       result.Bio,
        Website:   result.Website,
        CreatedAt: result.CreatedAt.Unix(),
        UpdatedAt: result.UpdatedAt.Unix(),
    }

    return connect.NewResponse(&authorv1.GetAuthorResponse{
        Author: author,
    }), nil
}
```

## 4. Create Connect Client (Outbound Adapter)

```go
// services/authsvc/internal/adapters/outbound/authorclient/connect/client.go
package connect

import (
    "context"
    "net/http"

    "connectrpc.com/connect"
    authorv1 "github.com/example/service-manager/contracts/author/v1"
    "github.com/example/service-manager/contracts/author/v1/authorconnect"
    "github.com/example/service-manager/services/authsvc/internal/application/ports"
)

type Client struct {
    client authorconnect.AuthorServiceClient
}

func NewClient(baseURL string, httpClient *http.Client) *Client {
    if httpClient == nil {
        httpClient = http.DefaultClient
    }

    client := authorconnect.NewAuthorServiceClient(
        httpClient,
        baseURL,
    )

    return &Client{client: client}
}

func (c *Client) GetAuthor(ctx context.Context, authorID string) (*ports.AuthorInfo, error) {
    req := connect.NewRequest(&authorv1.GetAuthorRequest{
        Id: authorID,
    })

    resp, err := c.client.GetAuthor(ctx, req)
    if err != nil {
        return nil, translateError(err)
    }

    author := resp.Msg.Author
    return &ports.AuthorInfo{
        ID:   author.Id,
        Name: author.Name,
        Bio:  author.Bio,
    }, nil
}

func translateError(err error) error {
    var connectErr *connect.Error
    if errors.As(err, &connectErr) {
        switch connectErr.Code() {
        case connect.CodeNotFound:
            return ports.ErrAuthorNotFound
        default:
            return ports.ErrAuthorServiceDown
        }
    }
    return err
}
```

## 5. Wire Based on Configuration

```go
// services/authsvc/cmd/authsvc/main.go
func main() {
    cfg := infra.LoadConfig()

    var authorClient ports.AuthorClient

    if cfg.UseInProcessBridge {
        // In-process via bridge
        authorServer := getAuthorServiceInprocServer()
        authorBridge := authorsvc.NewInprocClient(authorServer)
        authorClient = inproc.NewClient(authorBridge)
    } else {
        // Network via Connect
        authorClient = connect.NewClient(cfg.AuthorServiceURL, &http.Client{
            Timeout: 5 * time.Second,
        })
    }

    // Wire the rest of the service
    deps := infra.InitializeDependencies(cfg, authorClient)
    // ...
}
```
