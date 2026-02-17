# Complete Protobuf Workflow

When you're ready to add network transport, follow this workflow.

## Service-Specific vs Global Generation

This architecture supports two generation strategies:

| Strategy | When to Use | Command | Config File |
|----------|-------------|---------|-------------|
| **Service-Specific** | Development, service builds | `buf generate --template buf.gen.author.yaml` | `buf.gen.<service>.yaml` |
| **Global** | CI, full rebuilds, releasing | `buf generate` | `buf.gen.yaml` |

**Service-Specific Benefits:**
- ✅ Faster builds (only regenerate what you need)
- ✅ Language selection (Go-only for backend, TS-only for frontend)
- ✅ Isolation (changes to other APIs don't affect your build)
- ✅ Clear dependencies (each service owns its generation config)

**When to Use Global:**
- CI/CD pipelines that build all services
- Before creating releases or tags
- When updating shared protobuf dependencies
- Initial setup or major refactoring

## 1. Define Service Contract

**File: `contracts/proto/author/v1/author.proto`**

```protobuf
syntax = "proto3";

package author.v1;

option go_package = "github.com/example/service-manager/contracts/go/author/v1;authorv1";

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

### Create Service-Specific Generation Config

For each service, create a `buf.gen.<service>.yaml` in the contracts directory:

**File: `contracts/buf.gen.author.yaml`**

```yaml
# contracts/buf.gen.author.yaml - Generate only Author API
version: v2
managed:
  enabled: false
inputs:
  - directory: proto/author   # Point to author proto directory only
plugins:
  # Go: protobuf types
  - remote: buf.build/protocolbuffers/go
    out: go/author            # → contracts/go/author/
    opt: paths=source_relative
  # Go: Connect RPC stubs
  - remote: buf.build/connectrpc/go
    out: go/author            # → contracts/go/author/
    opt: paths=source_relative
```

### Create Per-Service Buf Module Config (if needed)

**File: `contracts/proto/author/buf.yaml`**

```yaml
version: v2
lint:
  use:
    - STANDARD
breaking:
  use:
    - FILE
```

### Run Generation

Run from the contracts directory using the service-specific template:

```bash
cd contracts
buf generate --template buf.gen.author.yaml

# Generated files (Go only, as configured):
# - go/author/v1/author.pb.go
# - go/author/v1/authorv1connect/author.connect.go
```

**Alternative: Global Generation (CI/Full Builds)**

```bash
cd contracts
buf generate  # Uses buf.gen.yaml, generates ALL APIs

# Generates code for all services in proto/:
# - go/author/v1/...
# - go/todo/v1/...
# - go/user/v1/...
# Plus TypeScript if configured
```

### Integration with Service Build Tools

Add generation task to your service's `mise.toml`:

```toml
# services/authorsvc/mise.toml
[tasks.generate]
description = "Generate code from protobuf definitions (Author API only)"
run = """
cd ../../contracts
buf generate --template buf.gen.author.yaml
"""

[tasks.build]
description = "Build the application binary"
depends = ["generate"]  # Auto-generate before building
run = "go build -o bin/authorsvc ./cmd/authorsvc"
```

Now `mise build` will automatically regenerate only the Author API before building.

## 3. Implement Connect Handler (Inbound Adapter)

```go
// services/authorsvc/internal/adapters/inbound/connect/handlers/author_handler.go
package handlers

import (
    "context"
    "connectrpc.com/connect"

    authorv1 "github.com/example/service-manager/contracts/go/author/v1"
    "github.com/example/service-manager/contracts/go/author/v1/authorconnect"
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
    authorv1 "github.com/example/service-manager/contracts/go/author/v1"
    "github.com/example/service-manager/contracts/go/author/v1/authorconnect"
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
