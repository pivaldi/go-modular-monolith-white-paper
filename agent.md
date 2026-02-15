# Agent Context: Go Modular Monolith Architecture

This repository follows a strict **Go Workspaces Modular Monolith** architecture using Hexagonal (Ports & Adapters) design.
When generating code, refactoring, or analyzing, you **MUST** adhere to these architectural rules.

## 1. Core Architecture Principles

**Structure:**
- Monorepo managed by `go.work` (workspace root)
- Each service is an independent Go module with its own `go.mod`
- Example: `services/serviceasvc/go.mod`, `services/servicebsvc/go.mod`

**The Golden Rule:**
> **Service modules MUST maintain independent dependency graphs.**
> Service A MUST NOT import Service B directly. Service A MUST NOT inherit Service B's dependencies.

**Communication:**
- Services communicate exclusively via **Bridge modules** (`bridge/<service>/`) for in-process calls
- Or via **Connect RPC** (`contracts/go/<service>/v1/`) for network calls
- Configuration determines which transport is used (same Port interface for both)

## 2. The "Pure Bridge" Pattern (Zero Dependencies)

The `bridge/<service>` directory defines the **public contract** for in-process communication.

### Strict Rules

**What MUST be in a bridge module:**
- Interface definitions (the "Port")
- DTOs (Plain structs with only standard library types)
- Public error variables (`var ErrEntityNotFound = errors.New(...)`)

**What MUST NOT be in a bridge module:**
- ❌ Any `require` statements in `go.mod` (zero external dependencies)
- ❌ Imports of any `internal/` packages
- ❌ Business logic, validation, or utility functions
- ❌ Database models, HTTP handlers, or infrastructure code
- ❌ Struct tags (`json`, `db`, `yaml`) - bridges are transport-agnostic

### Bridge Module Structure

```
bridge/serviceasvc/
├── go.mod              # MUST have zero requires (only module and go directives)
├── api.go              # Interface definition
├── dto.go              # Plain structs (no tags, no methods)
├── errors.go           # Public errors (var Err... = errors.New(...))
└── inproc_client.go    # Thin wrapper accepting the interface
```

**Key Insight:** The bridge is a **compile-time contract**, not a runtime artifact. It ensures type safety without coupling.

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
│   │   └── bridge/          # In-process (implements bridge interface)
│   │       └── inproc_server.go
│   │
│   └── outbound/        # Secondary adapters (driven by app)
│       ├── persistence/postgres/
│       ├── cache/redis/
│       └── servicebclient/
│           ├── inproc/      # In-process adapter (uses bridge)
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
- ❌ Bridge modules importing anything except standard library

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

### Implementation: In-Process (Bridge)

```go
// services/servicebsvc/internal/adapters/outbound/serviceaclient/inproc/client.go
type Client struct {
    bridge *serviceasvc.InprocClient  // from bridge/serviceasvc
}

func (c *Client) GetEntityA(ctx context.Context, id string) (*EntityA, error) {
    dto, err := c.bridge.GetEntityA(ctx, id)
    // Map bridge DTO → application DTO
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
        bridge := serviceasvc.NewInprocClient(server)
        serviceAClient = inproc.NewClient(bridge)
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
Example: `services/serviceasvc/test/contract/bridge_test.go` verifies serviceasvc implements `bridge/serviceasvc.ServiceA` correctly.

## 7. Code Generation & Buf Configuration

### Three buf config files, three purposes:

| File | Location | Purpose |
|------|----------|---------|
| `buf.yaml` | Repo root | Workspace config for `buf lint` and `buf breaking` in CI |
| `buf.yaml` | `contracts/proto/` | Module config: lint rules, breaking change rules |
| `buf.gen.yaml` | `contracts/` | Code generation: plugins, output paths |

### Generation workflow:

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
- Add dependencies to bridge modules (`bridge/*/go.mod` must have zero requires)
- Put business logic in DTOs
- Use `interface{}` or `any` in bridge interfaces (be explicit)
- Create cyclic dependencies (use `tools/arch-test/` to detect)
- Use tags in bridge DTOs (`json:`, `db:`, `yaml:` - bridges are transport-agnostic)

## 10. Migration Path

**Start Simple:**
1. Services in same process, bridge communication
2. Add protobuf contracts when ready (but keep using in-process)
3. Implement Connect handlers (test both transports)
4. Implement Connect clients
5. Deploy separately via config (`use_in_process: false`)

**Key Insight:** The architecture supports gradual extraction. Start monolith, extract services later with zero application code changes.

## 11. Quick Reference

**When adding a new service:**
1. Create `services/<newsvc>/go.mod`
2. Create `bridge/<newsvc>/` with zero dependencies
3. Implement hexagonal layers in `services/<newsvc>/internal/`
4. Add bridge implementation in `internal/adapters/inbound/bridge/`
5. Wire in `cmd/monolith/main.go`

**When service A needs data from service B:**
1. Define port interface in A: `internal/application/ports/serviceb_client.go`
2. Implement adapter in A: `internal/adapters/outbound/servicebclient/inproc/client.go`
3. Adapter uses B's bridge: `bridge/servicebsvc.ServiceB` interface
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
- `bridge-module-pattern.md` - In-depth bridge pattern explanation
- `complete-directory-structure.md` - Full directory layout with examples
- `testing-strategy.md` - Comprehensive testing guidance
- `arch-test.md` - Architectural boundary enforcement tooling
