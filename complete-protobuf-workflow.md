# Complete Protobuf Workflow

When you're ready to add network transport, follow this workflow.

## 1. Define Service Contract

**File: `contracts/proto/serviceasvc/v1/serviceasvc.proto`**

```protobuf
syntax = "proto3";

package serviceasvc.v1;

option go_package = "github.com/example/service-manager/contracts/gen/serviceasvc/v1;serviceasvcv1";

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

`buf.gen.yaml` lives in `contracts/` (not the repo root). Run generation from there:

```bash
cd contracts
buf generate

# Generated files (Go):
# - gen/serviceasvc/v1/serviceasvc.pb.go
# - gen/serviceasvc/v1/serviceasvcconnect/serviceasvc.connect.go

# Generated files (TypeScript, if ts plugins are configured):
# - gen/ts/serviceasvc/v1/serviceasvc_pb.ts
# - gen/ts/serviceasvc/v1/serviceasvc_connect.ts
```

## 3. Implement Connect Handler (Inbound Adapter)

```go
// services/serviceasvc/internal/adapters/inbound/connect/handlers/servicea_handler.go
package handlers

import (
    "context"
    "connectrpc.com/connect"

    serviceasvcv1 "github.com/example/service-manager/contracts/gen/serviceasvc/v1"
    "github.com/example/service-manager/contracts/gen/serviceasvc/v1/serviceasvcconnect"
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
var _ serviceasvcconnect.ServiceAServiceHandler = (*ServiceAHandler)(nil)

func (h *ServiceAHandler) GetServiceA(
    ctx context.Context,
    req *connect.Request[serviceasvcv1.GetServiceARequest],
) (*connect.Response[serviceasvcv1.GetServiceAResponse], error) {
    // Call application layer
    result, err := h.getServiceAQuery.Execute(ctx, req.Msg.Id)
    if err != nil {
        return nil, connect.NewError(connect.CodeNotFound, err)
    }

    // Map to protobuf
    servicea := &serviceasvcv1.ServiceA{
        Id:        result.ID,
        Name:      result.Name,
        Bio:       result.Bio,
        Website:   result.Website,
        CreatedAt: result.CreatedAt.Unix(),
        UpdatedAt: result.UpdatedAt.Unix(),
    }

    return connect.NewResponse(&serviceasvcv1.GetServiceAResponse{
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
    serviceasvcv1 "github.com/example/service-manager/contracts/gen/serviceasvc/v1"
    "github.com/example/service-manager/contracts/gen/serviceasvc/v1/serviceasvcconnect"
    "github.com/example/service-manager/services/servicebsvc/internal/application/ports"
)

type Client struct {
    client serviceasvcconnect.ServiceAServiceClient
}

func NewClient(baseURL string, httpClient *http.Client) *Client {
    if httpClient == nil {
        httpClient = http.DefaultClient
    }

    client := serviceasvcconnect.NewServiceAServiceClient(
        httpClient,
        baseURL,
    )

    return &Client{client: client}
}

func (c *Client) GetServiceA(ctx context.Context, serviceaID string) (*ports.ServiceAInfo, error) {
    req := connect.NewRequest(&serviceasvcv1.GetServiceARequest{
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
