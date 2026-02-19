# Agent Context: Go Modular Monolith Architecture

This repository follows a strict **Go Workspaces Modular Monolith** architecture using Hexagonal (Ports & Adapters) design.
When generating code, refactoring, or analyzing, you **MUST** adhere to these architectural rules.

# Directory Structure

```
monolith/
├── go.work                    # Go workspace definition
│                              # use (./contracts ./contracts/definitions/serviceasvc ./services/...)
│
├── mise.toml                  # Root orchestration tasks
│
├── buf.yaml                   # Buf workspace config (lint/breaking rules across all modules)
│
├── contracts/                # Unified contracts module
│   ├── go.mod                # Module: github.com/example/service-manager/contracts
│   ├── buf.gen.yaml          # Code generation config (run `buf generate` from here)
│   ├── README.md             # Contract versioning strategy
│   │
│   ├── definitions/          # Contract Definitions (pure Go interfaces)
│   │   │                     # ZERO dependencies - truly dependency-free
│   │   │
│   │   ├── serviceasvc/      # Service A public API
│   │   │   ├── go.mod        # Module: contracts/definitions/serviceasvc v1.0.0
│   │   │   │                 # Dependencies: ZERO (truly dependency-free)
│   │   │   │
│   │   │   ├── README.md     # Contract usage documentation
│   │   │   │
│   │   │   ├── api.go        # Public service interface
│   │   │   │   # type ServiceAService interface {
│   │   │   │   #   GetServiceA(ctx, id) (*ServiceADTO, error)
│   │   │   │   #   CreateServiceA(ctx, req) (*ServiceADTO, error)
│   │   │   │   # }
│   │   │   │
│   │   │   ├── dto.go        # Domain-friendly DTOs
│   │   │   │   # type ServiceADTO struct { ID, Name, Bio string }
│   │   │   │
│   │   │   ├── errors.go     # Public error types
│   │   │   │   # var ErrServiceANotFound = errors.New("entity not found")
│   │   │   │
│   │   │   └── inproc_client.go  # In-process client (thin wrapper)
│   │   │       # Implements ServiceAService interface
│   │   │       # Accepts ServiceAService interface (not concrete type!)
│   │   │       # Calls via interface (function call, no network)
│   │   │
│   │   └── servicebsvc/      # Service B public API (similar structure)
│   │       ├── go.mod
│   │       ├── api.go
│   │       ├── dto.go
│   │       ├── errors.go
│   │       └── inproc_client.go
│   │
│   ├── proto/                # CLEAN: Protobuf schemas only (OPTIONAL - for network transport)
│   │   ├── buf.yaml          # Buf module config (marks this as the proto root)
│   │   ├── serviceasvc/v1/
│   │   │   └── serviceasvc.proto   # Service A protobuf definition
│   │   └── servicebsvc/v1/
│   │       └── servicebsvc.proto   # Service B protobuf definition
│   │
│   └── gen/                  # DIRTY: Generated code from protobuf (do not edit manually)
│       ├── serviceasvc/v1/
│       │   ├── serviceasvc.pb.go          # Generated protobuf types
│       │   └── serviceasvcconnect/
│       │       └── serviceasvc.connect.go # Generated Connect stubs
│       │
│       ├── servicebsvc/v1/
│       │   ├── servicebsvc.pb.go          # Generated protobuf types
│       │   └── servicebsvcconnect/
│       │       └── servicebsvc.connect.go # Generated Connect stubs
│       │
│       └── ts/               # Generated TypeScript code
│           ├── package.json  # npm package: @example/contracts
│           ├── serviceasvc/v1/
│           │   ├── serviceasvc_pb.ts        # Generated protobuf types
│           │   └── serviceasvc_connect.ts   # Generated Connect stubs
│           └── servicebsvc/v1/
│               ├── servicebsvc_pb.ts        # Generated
│               └── servicebsvc_connect.ts   # Generated
│
├── services/
│   │
│   ├── servicebsvc/              # Service B
│   │   ├── go.mod            # Module: services/servicebsvc v2.1.0
│   │   │                     # Dependencies: contracts/definitions/serviceasvc, contracts (optional)
│   │   │
│   │   ├── README.md         # Service documentation
│   │   ├── mise.toml         # Service-specific tasks
│   │   ├── Dockerfile
│   │   │
│   │   ├── cmd/
│   │   │   └── servicebsvc/
│   │   │       └── main.go   # Composition root
│   │   │           # Wires dependencies:
│   │   │           # - In dev: uses serviceasvc.InprocClient
│   │   │           # - In prod: uses serviceasvcconnect.Client
│   │   │
│   │   ├── test/             # Service-level tests
│   │   │   └── contract/     # CONTRACT TESTS (verify servicebsvc's own API contracts)
│   │   │       └── http_api_test.go  # Tests servicebsvc HTTP responses match spec
│   │   │
│   │   └── internal/         # PRIVATE - cannot be imported by other services
│   │       │
│   │       ├── domain/       # DOMAIN LAYER (pure business logic)
│   │       │   │             # UNIT TESTS (co-located *_test.go)
│   │       │   │             # No external dependencies
│   │       │   │             # No adapters, no infrastructure
│   │       │   │
│   │       │   ├── user/     # User aggregate
│   │       │   │   ├── user.go           # Entity (aggregate root)
│   │       │   │   ├── user_test.go      # UNIT TEST
│   │       │   │   ├── email.go          # Value object
│   │       │   │   ├── email_test.go     # UNIT TEST
│   │       │   │   ├── password.go       # Value object
│   │       │   │   ├── password_test.go  # UNIT TEST
│   │       │   │   └── repository.go     # Repository interface (port)
│   │       │   │
│   │       │   ├── session/  # Session aggregate
│   │       │   │   ├── session.go        # Entity
│   │       │   │   ├── session_test.go   # UNIT TEST
│   │       │   │   ├── token.go          # Value object
│   │       │   │   └── repository.go     # Repository interface
│   │       │   │
│   │       │   └── errors.go # Domain errors
│   │       │
│   │       ├── application/  # APPLICATION LAYER (use cases)
│   │       │   │             # UNIT TESTS (co-located *_test.go with mocked ports)
│   │       │   │             # Orchestrates domain objects
│   │       │   │             # Depends on: domain, ports (interfaces only)
│   │       │   │
│   │       │   ├── command/  # Write operations
│   │       │   │   ├── login.go          # Login use case
│   │       │   │   ├── login_test.go     # UNIT TEST (mocked ports)
│   │       │   │   ├── logout.go         # Logout use case
│   │       │   │   ├── register.go       # Registration use case
│   │       │   │   ├── register_test.go  # UNIT TEST (mocked ports)
│   │       │   │   └── change_password.go
│   │       │   │
│   │       │   ├── query/    # Read operations
│   │       │   │   ├── get_user.go
│   │       │   │   ├── get_user_test.go  # UNIT TEST (mocked ports)
│   │       │   │   └── validate_token.go
│   │       │   │
│   │       │   ├── dto/      # Application DTOs
│   │       │   │   ├── user_dto.go
│   │       │   │   └── session_dto.go
│   │       │   │
│   │       │   └── ports/    # APPLICATION PORTS (interfaces)
│   │       │       │         # Owned by application layer
│   │       │       │         # Implemented by adapters
│   │       │       │
│   │       │       ├── servicea_client.go  # Outbound port
│   │       │       │   # type ServiceAClient interface {
│   │       │       │   #   GetEntityA(ctx, id) (*EntityA, error)
│   │       │       │   # }
│   │       │       │
│   │       │       ├── cache.go          # Outbound port
│   │       │       ├── logger.go         # Outbound port
│   │       │       └── event_publisher.go
│   │       │
│   │       ├── adapters/     # ADAPTERS LAYER (I/O boundaries)
│   │       │   │             # INTEGRATION TESTS (co-located *_test.go with real infrastructure)
│   │       │   │             # Implements ports from application layer
│   │       │   │
│   │       │   ├── inbound/  # Inbound adapters (primary/driving)
│   │       │   │   │         # External world -> Application
│   │       │   │   │
│   │       │   │   ├── http/ # HTTP REST adapter
│   │       │   │   │   ├── server.go
│   │       │   │   │   ├── server_test.go        # INTEGRATION TEST (real HTTP)
│   │       │   │   │   ├── handlers/
│   │       │   │   │   │   ├── login.go
│   │       │   │   │   │   ├── login_test.go     # INTEGRATION TEST
│   │       │   │   │   │   ├── logout.go
│   │       │   │   │   │   └── register.go
│   │       │   │   │   ├── middleware/
│   │       │   │   │   └── dto/
│   │       │   │   │
│   │       │   │   └── connect/  # Connect/gRPC adapter (optional)
│   │       │   │       ├── server.go
│   │       │   │       └── handlers/
│   │       │   │           └── serviceb_handler.go
│   │       │   │
│   │       │   └── outbound/ # Outbound adapters (secondary/driven)
│   │       │       │         # Application -> External systems
│   │       │       │
│   │       │       ├── persistence/  # Database adapters
│   │       │       │   ├── postgres/
│   │       │       │   │   ├── user_repository.go
│   │       │       │   │   ├── user_repository_test.go  # INTEGRATION TEST (real DB)
│   │       │       │   │   ├── session_repository.go
│   │       │       │   │   ├── session_repository_test.go  # INTEGRATION TEST
│   │       │       │   │   └── migrations/
│   │       │       │   │
│   │       │       │   └── memory/   # In-memory (testing)
│   │       │       │
│   │       │       ├── serviceaclient/  # Service A client adapters
│   │       │       │   ├── inproc/   # In-process adapter (TODAY)
│   │       │       │   │   ├── client.go
│   │       │       │   │   │   # Implements ports.ServiceAClient
│   │       │       │   │   │   # Uses serviceasvc.InprocClient
│   │       │       │   │   │   # Zero network overhead
│   │       │       │   │   └── client_test.go     # INTEGRATION TEST
│   │       │       │   │
│   │       │       │   └── connect/  # Network adapter (LATER)
│   │       │       │       ├── client.go
│   │       │       │       │   # Implements ports.ServiceAClient
│   │       │       │       │   # Uses serviceasvcconnect.Client
│   │       │       │       │   # Network overhead (HTTP, serialization)
│   │       │       │       └── client_test.go     # INTEGRATION TEST
│   │       │       │
│   │       │       ├── cache/
│   │       │       │   ├── redis/
│   │       │       │   │   ├── cache.go
│   │       │       │   │   └── cache_test.go      # INTEGRATION TEST (real Redis)
│   │       │       │   └── memory/
│   │       │       │
│   │       │       └── logger/
│   │       │           └── zap/
│   │       │
│   │       └── infra/        # INFRASTRUCTURE LAYER
│   │           │             # Technical concerns, wiring
│   │           │
│   │           ├── config/   # Configuration management
│   │           ├── wire/     # Dependency injection (optional)
│   │           ├── database/ # DB connection management
│   │           ├── observability/
│   │           └── server/   # Server lifecycle
│   │
│   └── serviceasvc/            # Service A (similar structure)
│       ├── go.mod            # Module: services/serviceasvc v1.0.0
│       │                     # Dependencies: contracts/definitions/serviceasvc (to implement server)
│       │
│       ├── cmd/serviceasvc/
│       │   └── main.go       # Wires serviceasvc.InprocServer to internal/application
│       │
│       ├── test/             # Service-level tests
│       │   └── contract/     # CONTRACT TESTS (verify serviceasvc implements contract correctly)
│       │       ├── contracts_test.go      # Tests InprocServer implements serviceasvc.ServiceAService
│       │       └── http_api_test.go       # Tests HTTP API matches spec
│       │
│       └── internal/
│           ├── domain/       # UNIT TESTS (co-located *_test.go)
│           │   └── entitya/   # Entity A aggregate
│           │       ├── entitya.go
│           │       ├── entitya_test.go    # UNIT TEST
│           │       ├── entitya_id.go
│           │       ├── name.go
│           │       ├── name_test.go      # UNIT TEST
│           │       └── repository.go
│           │
│           ├── application/  # UNIT TESTS (co-located *_test.go with mocked ports)
│           │   ├── command/
│           │   │   ├── create_entitya.go
│           │   │   ├── create_entitya_test.go  # UNIT TEST (mocked ports)
│           │   │   └── update_entitya.go
│           │   │
│           │   ├── query/
│           │   │   ├── get_entitya.go
│           │   │   ├── get_entitya_test.go     # UNIT TEST (mocked ports)
│           │   │   └── list_entitya.go
│           │   │
│           │   └── ports/
│           │       ├── image_client.go
│           │       └── logger.go
│           │
│           ├── adapters/     # INTEGRATION TESTS (co-located *_test.go with real infrastructure)
│           │   ├── inbound/      # Inbound adapters (primary/driving)
│           │   │   │
│           │   │   ├── contracts/   # Contract adapter (in-process)
│           │   │   │   ├── inproc_server.go
│           │   │   │   │   # Implements serviceasvc.ServiceAService interface
│           │   │   │   │   # Wraps serviceasvc/internal/application
│           │   │   │   │   # CAN import serviceasvc/internal (same module!)
│           │   │   │   │   # Returns ServiceAService interface for loose coupling
│           │   │   │   └── inproc_server_test.go  # INTEGRATION TEST
│           │   │   │
│           │   │   ├── http/     # HTTP REST adapter
│           │   │   │   ├── server.go
│           │   │   │   └── server_test.go         # INTEGRATION TEST
│           │   │   │
│           │   │   └── connect/  # Connect/gRPC adapter
│           │   │       └── ...
│           │   │
│           │   └── outbound/
│           │       ├── persistence/postgres/
│           │       │   ├── entitya_repository.go
│           │       │   └── entitya_repository_test.go  # INTEGRATION TEST (real DB)
│           │       ├── imageclient/
│           │       └── cache/
│           │
│           └── infra/
│
├── tools/
│   └── arch-test/            # Architectural boundary enforcement
│       └── main.go           # Validates import rules in CI
│
├── docs/
│   ├── architecture/
│   │   ├── decisions/        # Architecture Decision Records
│   │   └── diagrams/
│   │
│   └── development/
│       ├── local-setup.md
│       ├── testing-guide.md
│       └── adding-a-service.md
│
└── test/                     # Cross-service tests (see testing-strategy.md)
    └── e2e/                  # E2E TESTS (full system, complete user journeys)
        ├── go.mod
        ├── user_journey_test.go      # E2E TEST
        ├── auth_flow_test.go         # E2E TEST
        └── fixtures/
            ├── docker-compose.yml
            └── seed_data.sql
```


## 1. Core Architecture Principles

**Structure:**
- Monorepo managed by `go.work` (workspace root)
- Each service is an independent Go module with its own `go.mod`
- Example: `services/serviceasvc/go.mod`, `services/servicebsvc/go.mod`

**The Golden Rule:**
> **Service modules MUST maintain independent dependency graphs.**
> Service A MUST NOT import Service B directly. Service A MUST NOT inherit Service B's dependencies.

**Communication:**
- Services communicate exclusively via **Contract Definitions** (`contracts/definitions/<service>/`) for in-process calls
- Or via **Connect RPC** (`contracts/proto/<service>/v1/`) for network calls
- Configuration determines which transport is used (same Port interface for both)

## 2. The "Pure Contract Definition" Pattern (Zero Dependencies)

The `contracts/definitions/<service>` directory defines the **public contract** for in-process communication.

### Strict Rules

**What MUST be in a contract definition module:**
- Interface definitions (the "Port")
- DTOs (Plain structs with only standard library types)
- Public error variables (`var ErrEntityNotFound = errors.New(...)`)

**What MUST NOT be in a contract definition module:**
- ❌ Any `require` statements in `go.mod` (zero external dependencies)
- ❌ Imports of any `internal/` packages
- ❌ Business logic, validation, or utility functions
- ❌ Database models, HTTP handlers, or infrastructure code
- ❌ Struct tags (`json`, `db`, `yaml`) - contract definitions are transport-agnostic

### Contract Definition Module Structure

```
contracts/definitions/serviceasvc/
├── go.mod              # MUST have zero requires (only module and go directives)
├── api.go              # Interface definition
├── dto.go              # Plain structs (no tags, no methods)
├── errors.go           # Public errors (var Err... = errors.New(...))
└── inproc_client.go    # Thin wrapper accepting the interface
```

**Key Insight:** The contract definition is a **compile-time contract**, not a runtime artifact. It ensures type safety without coupling.

## 3. Hexagonal Architecture (Ports & Adapters)

Services use strict hexagonal architecture with these layers:

```
services/serviceasvc/internal/
├── domain/              # LAYER 1: Pure business logic
│   ├── entitya/         # Aggregates, entities, value objects
│   │   ├── entitya.go       # Entity (aggregate root)
│   │   ├── name.go          # Value object
│   │   └── repository.go    # Repository interface (Port)
│   └── errors.go        # Domain errors
│
├── application/         # LAYER 2: Use cases (orchestration)
│   ├── command/         # Write operations
│   │   └── create_entitya.go
│   ├── query/           # Read operations
│   │   └── get_entitya.go
│   ├── dto/             # Application-specific DTOs
│   └── ports/           # Port interfaces (owned by application)
│       ├── serviceb_client.go   # Outbound port
│       ├── cache.go
│       └── logger.go
│
├── adapters/            # LAYER 3: I/O boundaries
│   ├── inbound/         # Primary adapters (drive the app)
│   │   ├── http/            # REST API
│   │   ├── connect/         # gRPC/Connect
│   │   └── contracts/       # In-process (implements contract interface)
│   │       └── inproc_server.go
│   │
│   └── outbound/        # Secondary adapters (driven by app)
│       ├── persistence/postgres/
│       ├── cache/redis/
│       └── servicebclient/
│           ├── inproc/      # In-process adapter (uses contract definition)
│           └── connect/     # Network adapter (uses Connect)
│
└── infra/               # LAYER 4: Infrastructure
    ├── config/          # Configuration structs
    ├── database/        # Connection management
    └── server/          # Server lifecycle
```

### Dependency Rules (Critical)

```
Domain → (nothing)
Application → Domain + Ports (interfaces only)
Adapters → Application + Domain
Infra → All layers (composition)
```

**Violations to watch for:**
- ❌ Domain importing application or adapters
- ❌ Application importing concrete adapters (only ports)
- ❌ Services importing other services' `internal/` packages
- ❌ Contract definition modules importing anything except standard library

## 4. Protobuf Contracts (Optional - For Network Transport)

When services need network communication, use the `contracts/` module.

### Directory Structure

```
contracts/
├── go.mod               # Module: github.com/example/repo/contracts
├── buf.gen.yaml         # Code generation config (run buf generate from here)
├── proto/               # CLEAN: .proto schemas only
│   ├── buf.yaml         # Buf module config
│   ├── servicea/v1/
│   │   └── servicea.proto
│   └── serviceb/v1/
│       └── serviceb.proto
├── go/                  # DIRTY: Generated Go code (do not edit)
│   ├── servicea/v1/
│   │   ├── servicea.pb.go
│   │   └── serviceaconnect/
│   │       └── servicea.connect.go
│   └── serviceb/v1/
└── ts/                  # DIRTY: Generated TypeScript code (do not edit)
    ├── package.json     # npm package: @example/contracts
    ├── servicea/v1/
    │   ├── servicea_pb.ts
    │   └── servicea_connect.ts
    └── serviceb/v1/
```

### Key Points

- **One module, multiple languages:** All generated code (Go, TypeScript) lives in one `contracts/` module for atomic versioning
- **Clear separation:** `.proto` schemas in `proto/`, generated code in language-specific subdirectories
- **buf.gen.yaml placement:** Lives at `contracts/buf.gen.yaml`, run `buf generate` from `contracts/` directory
- **Imports:** Services import `github.com/example/repo/contracts/go/servicea/v1`

## 5. Transport Swapping (In-Process ↔ Network)

The same Port interface supports both transports:

```go
// Application Port (owned by servicebsvc)
type ServiceAClient interface {
    GetEntityA(ctx context.Context, id string) (*EntityA, error)
}
```

### Implementation: In-Process (Contract Definition)

```go
// services/servicebsvc/internal/adapters/outbound/serviceaclient/inproc/client.go
type Client struct {
    contractClient *serviceasvc.InprocClient  // from contracts/definitions/serviceasvc
}

func (c *Client) GetEntityA(ctx context.Context, id string) (*EntityA, error) {
    dto, err := c.contractClient.GetEntityA(ctx, id)
    // Map contract DTO → application DTO
    return &EntityA{ID: dto.ID, Name: dto.Name}, err
}
```

### Implementation: Network (Connect)

```go
// services/servicebsvc/internal/adapters/outbound/serviceaclient/connect/client.go
type Client struct {
    client serviceaconnect.ServiceAClient  // from contracts/go/servicea/v1
}

func (c *Client) GetEntityA(ctx context.Context, id string) (*EntityA, error) {
    req := connect.NewRequest(&serviceav1.GetEntityARequest{Id: id})
    resp, err := c.client.GetEntityA(ctx, req)
    // Map protobuf → application DTO
    return &EntityA{ID: resp.Msg.Entity.Id, Name: resp.Msg.Entity.Name}, err
}
```

### Wiring (Composition Root)

```go
// cmd/monolith/main.go
func main() {
    cfg := loadConfig()

    var serviceAClient ports.ServiceAClient

    if cfg.UseInProcess {
        // In-process: <1μs latency, zero serialization
        server := getServiceAInprocServer()
        contractClient := serviceasvc.NewInprocClient(server)
        serviceAClient = inproc.NewClient(contractClient)
    } else {
        // Network: 1-5ms latency, protobuf serialization
        serviceAClient = connect.NewClient(cfg.ServiceAURL, httpClient)
    }

    // Application layer doesn't know the difference
    app := application.New(serviceAClient, ...)
}
```

## 6. Testing Strategy

### Unit Tests
- **Location:** Co-located with code (`*_test.go`)
- **Domain:** Test business logic with zero dependencies
- **Application:** Test use cases with mocked ports (interfaces)
- **No real infrastructure:** Use mocks/stubs

### Integration Tests
- **Location:** Co-located in `internal/adapters/` (`*_test.go`)
- **Adapters:** Test with real infrastructure (testcontainers for DB, real Redis)
- **Focus:** Verify adapter correctly implements port interface

### Contract Tests
- **Location:** `services/<svc>/test/contract/`
- **Purpose:** Verify service correctly implements its public API
- **Provider responsibility:** Each service tests its OWN contract implementation

### E2E Tests
- **Location:** Root-level `test/e2e/`
- **Purpose:** Complete user journeys across all services
- **Infrastructure:** docker-compose with all services running

**Key Rule:** Contract tests live in the **provider** service, not the consumer.
Example: `services/serviceasvc/test/contract/contracts_test.go` verifies serviceasvc implements `contracts/definitions/serviceasvc.ServiceA` correctly.

## 7. Code Generation & Buf Configuration

### Three buf config files, three purposes:

| File | Location | Purpose |
|------|----------|---------|
| `buf.yaml` | Repo root | Workspace config for `buf lint` and `buf breaking` in CI |
| `buf.yaml` | `contracts/proto/` | Module config: lint rules, breaking change rules |
| `buf.gen.yaml` | `contracts/` | Code generation: plugins, output paths |

### Generation workflow:

```yaml
# contracts/buf.gen.yaml example
version: v2
inputs:
  - directory: proto
plugins:
  - remote: buf.build/protocolbuffers/go
    out: go
    opt: paths=source_relative
  - remote: buf.build/connectrpc/go
    out: go
    opt: paths=source_relative
  - remote: buf.build/bufbuild/es
    out: ts
    opt: target=ts
  - remote: buf.build/connectrpc/es
    out: ts
    opt: target=ts
```

Run generation:

```bash
cd contracts
buf generate

# Outputs:
# - go/servicea/v1/servicea.pb.go
# - go/servicea/v1/serviceaconnect/servicea.connect.go
# - ts/servicea/v1/servicea_pb.ts
# - ts/servicea/v1/servicea_connect.ts
```

## 8. Wiring & Dependency Injection

**Composition Root:** `cmd/monolith/main.go` (or `cmd/monolith/app/`)
- Instantiate all services
- Wire dependencies via constructor injection
- Services expose `New(deps...)` functions
- Use `errgroup` for concurrent service startup

**Anti-patterns:**
- ❌ No root-level `bootstrap/` directory
- ❌ No `common/` or `shared/` module (prefer duplication over coupling)
- ❌ No service locator pattern
- ❌ No global singletons

## 9. Prohibited Patterns

**DO NOT:**
- Create `pkg/` or `shared/` modules (duplicates 50 lines of code, saves 5 services from dependency hell)
- Put generated protobuf code in `services/<svc>/internal/`
- Import `services/servicea/internal/...` from another service
- Add dependencies to contract definition modules (`contracts/definitions/*/go.mod` must have zero requires)
- Put business logic in DTOs
- Use `interface{}` or `any` in contract interfaces (be explicit)
- Create cyclic dependencies (use `tools/arch-test/` to detect)
- Use tags in contract DTOs (`json:`, `db:`, `yaml:` - contract definitions are transport-agnostic)

## 10. Migration Path

**Start Simple:**
1. Services in same process, contract-based communication
2. Add protobuf contracts when ready (but keep using in-process)
3. Implement Connect handlers (test both transports)
4. Implement Connect clients
5. Deploy separately via config (`use_in_process: false`)

**Key Insight:** The architecture supports gradual extraction. Start monolith, extract services later with zero application code changes.

## 11. Quick Reference

**When adding a new service:**
1. Create `services/<newsvc>/go.mod`
2. Create `contracts/definitions/<newsvc>/` with zero dependencies
3. Implement hexagonal layers in `services/<newsvc>/internal/`
4. Add contract implementation in `internal/adapters/inbound/contracts/`
5. Wire in `cmd/monolith/main.go`

**When service A needs data from service B:**
1. Define port interface in A: `internal/application/ports/serviceb_client.go`
2. Implement adapter in A: `internal/adapters/outbound/servicebclient/inproc/client.go`
3. Adapter uses B's contract definition: `contracts/definitions/servicebsvc.ServiceB` interface
4. Wire in composition root

**When adding network transport:**
1. Create `.proto` in `contracts/proto/<svc>/v1/`
2. Run `buf generate` from `contracts/`
3. Implement Connect handler in provider: `internal/adapters/inbound/connect/`
4. Implement Connect client in consumer: `internal/adapters/outbound/<provider>/connect/`
5. Update composition root to choose transport via config

## 12. Further Reading

See these files in the repository root:
- `protobuf-contracts.md` - Detailed protobuf and Connect patterns
- `contract-definition-pattern.md` - In-depth contract-based architecture explanation
- `complete-directory-structure.md` - Full directory layout with examples
- `testing-strategy.md` - Comprehensive testing guidance
- `arch-test.md` - Architectural boundary enforcement tooling
