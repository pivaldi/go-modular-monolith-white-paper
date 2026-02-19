# Complete Protobuf Workflow

When you're ready to add network transport, follow this workflow.

## Service-Specific vs Global Generation

This architecture supports two generation strategies:

| Strategy | When to Use | Command | Config File |
|----------|-------------|---------|-------------|
| **Service-Specific** | Development, service builds | `buf generate --template buf.gen.servicea.yaml` | `buf.gen.<service>.yaml` |
| **Global** | CI, full rebuilds, releasing | `buf generate` | `buf.gen.yaml` |

**Service-Specific Benefits:**
- ✓ Faster builds (only regenerate what you need)
- ✓ Language selection (Go-only for backend, TS-only for frontend)
- ✓ Isolation (changes to other APIs don't affect your build)
- ✓ Clear dependencies (each service owns its generation config)

**When to Use Global:**
- CI/CD pipelines that build all services
- Before creating releases or tags
- When updating shared protobuf dependencies
- Initial setup or major refactoring

## 1. Define Service Contract

**File: `contracts/proto/servicea/v1/servicea.proto`**

```protobuf
syntax = "proto3";

package servicea.v1;

option go_package = "github.com/example/service-manager/contracts/gen/go/servicea/v1;serviceav1";

service ServiceAService {
  rpc GetServiceA(GetServiceARequest) returns (GetServiceAResponse) {}
  rpc CreateServiceA(CreateServiceARequest) returns (CreateServiceAResponse) {}
}

message ServiceA {
  string id = 1;
  string name = 2;
  string bio = 3;
  string website = 4;
  int64 created_at = 5;
  int64 updated_at = 6;
}

message GetServiceARequest {
  string id = 1;
}

message GetServiceAResponse {
  ServiceA servicea = 1;
}

message CreateServiceARequest {
  string name = 1;
  string bio = 2;
  string website = 3;
}

message CreateServiceAResponse {
  ServiceA servicea = 1;
}
```

## 2. Generate Code

### Create Service-Specific Generation Config

For each service, create a `buf.gen.<service>.yaml` in the contracts directory:

**File: `contracts/buf.gen.servicea.yaml`**

```yaml
# contracts/buf.gen.servicea.yaml - Generate only Service A API
version: v2
managed:
  enabled: false
inputs:
  - directory: .
    paths:
      - proto/servicea  # Filter to only Service A protos
plugins:
  # Go: protobuf types
  - remote: buf.build/protocolbuffers/go
    out: gen/go
    opt: paths=source_relative
  # Go: Connect RPC stubs
  - remote: buf.build/connectrpc/go
    out: gen/go
    opt: paths=source_relative
```

### Create Per-Service Buf Module Config (if needed)

**File: `contracts/proto/servicea/buf.yaml`**

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
buf generate --template buf.gen.servicea.yaml

# Generated files (Go):
# - gen/go/servicea/v1/servicea.pb.go
# - gen/go/servicea/v1/serviceav1connect/servicea.connect.go

# Generated files (TypeScript, if ts plugins are configured):
# - gen/ts/servicea/v1/servicea_pb.ts
# - gen/ts/servicea/v1/servicea_connect.ts
```

**Alternative: Global Generation (CI/Full Builds)**

```bash
cd contracts
buf generate  # Uses buf.gen.yaml, generates ALL APIs

# Generates code for all services in proto/:
# - gen/go/servicea/v1/...
# - gen/go/serviceb/v1/...
# - gen/ts/servicea/v1/...
# - gen/ts/serviceb/v1/...
```

### Integration with Service Build Tools

Add generation task to your service's `mise.toml`:

```toml
# services/serviceasvc/mise.toml
[tasks.generate]
description = "Generate code from protobuf definitions (Service A API only)"
run = """
cd ../../contracts
buf generate --template buf.gen.servicea.yaml
"""

[tasks.build]
description = "Build the application binary"
depends = ["generate"]  # Auto-generate before building
run = "go build -o bin/serviceasvc ./cmd/serviceasvc"
```

Now `mise build` will automatically regenerate only the Service A API before building.

## 3. Implement Connect Handler (Inbound Adapter)

```go
// services/serviceasvc/internal/adapters/inbound/connect/handlers/servicea_handler.go
package handlers

import (
    "context"
    "connectrpc.com/connect"

    serviceav1 "github.com/example/service-manager/contracts/gen/go/servicea/v1"
    "github.com/example/service-manager/contracts/gen/go/servicea/v1/serviceav1connect"
    "github.com/example/service-manager/services/serviceasvc/internal/application/command"
    "github.com/example/service-manager/services/serviceasvc/internal/application/query"
)

type ServiceAHandler struct {
    getServiceAQuery  *query.GetServiceAQuery
    createServiceACmd *command.CreateServiceACommand
}

func NewServiceAHandler(
    getServiceAQuery *query.GetServiceAQuery,
    createServiceACmd *command.CreateServiceACommand,
) *ServiceAHandler {
    return &ServiceAHandler{
        getServiceAQuery:  getServiceAQuery,
        createServiceACmd: createServiceACmd,
    }
}

// Ensure we implement the interface
var _ serviceav1connect.ServiceAServiceHandler = (*ServiceAHandler)(nil)

func (h *ServiceAHandler) GetServiceA(
    ctx context.Context,
    req *connect.Request[serviceav1.GetServiceARequest],
) (*connect.Response[serviceav1.GetServiceAResponse], error) {
    // Call application layer
    result, err := h.getServiceAQuery.Execute(ctx, req.Msg.Id)
    if err != nil {
        return nil, connect.NewError(connect.CodeNotFound, err)
    }

    // Map to protobuf
    servicea := &serviceav1.ServiceA{
        Id:        result.ID,
        Name:      result.Name,
        Bio:       result.Bio,
        Website:   result.Website,
        CreatedAt: result.CreatedAt.Unix(),
        UpdatedAt: result.UpdatedAt.Unix(),
    }

    return connect.NewResponse(&serviceav1.GetServiceAResponse{
        Servicea: servicea,
    }), nil
}
```

## 4. Create Connect Client (Outbound Adapter)

```go
// services/servicebsvc/internal/adapters/outbound/serviceaclient/connect/client.go
package connect

import (
    "context"
    "net/http"

    "connectrpc.com/connect"
    serviceav1 "github.com/example/service-manager/contracts/gen/go/servicea/v1"
    "github.com/example/service-manager/contracts/gen/go/servicea/v1/serviceav1connect"
    "github.com/example/service-manager/services/servicebsvc/internal/application/ports"
)

type Client struct {
    client serviceav1connect.ServiceAServiceClient
}

func NewClient(baseURL string, httpClient *http.Client) *Client {
    if httpClient == nil {
        httpClient = http.DefaultClient
    }

    client := serviceav1connect.NewServiceAServiceClient(
        httpClient,
        baseURL,
    )

    return &Client{client: client}
}

func (c *Client) GetServiceA(ctx context.Context, serviceaID string) (*ports.ServiceAInfo, error) {
    req := connect.NewRequest(&serviceav1.GetServiceARequest{
        Id: serviceaID,
    })

    resp, err := c.client.GetServiceA(ctx, req)
    if err != nil {
        return nil, translateError(err)
    }

    servicea := resp.Msg.Servicea
    return &ports.ServiceAInfo{
        ID:   servicea.Id,
        Name: servicea.Name,
        Bio:  servicea.Bio,
    }, nil
}

func translateError(err error) error {
    var connectErr *connect.Error
    if errors.As(err, &connectErr) {
        switch connectErr.Code() {
        case connect.CodeNotFound:
            return ports.ErrServiceANotFound
        default:
            return ports.ErrServiceAServiceDown
        }
    }
    return err
}
```

## 5. Wire Based on Configuration

```go
// services/servicebsvc/cmd/servicebsvc/main.go
func main() {
    cfg := infra.LoadConfig()

    var serviceaClient ports.ServiceAClient

    if cfg.UseInProcessContracts {
        // In-process via contract definition
        serviceaServer := getServiceAServiceInprocServer()
        serviceaContract := serviceasvc.NewInprocClient(serviceaServer)
        serviceaClient = inproc.NewClient(serviceaContract)
    } else {
        // Network via Connect
        serviceaClient = connect.NewClient(cfg.ServiceAServiceURL, &http.Client{
            Timeout: 5 * time.Second,
        })
    }

    // Wire the rest of the service
    deps := infra.InitializeDependencies(cfg, serviceaClient)
    // ...
}
```
