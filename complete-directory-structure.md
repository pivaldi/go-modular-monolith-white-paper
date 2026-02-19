# Complete Directory Structure

```
service-manager/
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
│   │   ├── servicea/v1/
│   │   │   └── servicea.proto   # Service A protobuf definition
│   │   └── serviceb/v1/
│   │       └── serviceb.proto   # Service B protobuf definition
│   │
│   └── gen/                  # DIRTY: Generated code from protobuf (do not edit manually)
│       ├── go/               # Go generated code (language first, then service)
│       │   ├── servicea/v1/
│       │   │   ├── servicea.pb.go          # Generated protobuf types
│       │   │   └── serviceav1connect/
│       │   │       └── servicea.connect.go # Generated Connect stubs
│       │   │
│       │   └── serviceb/v1/
│       │       ├── serviceb.pb.go          # Generated protobuf types
│       │       └── servicebv1connect/
│       │           └── serviceb.connect.go # Generated Connect stubs
│       │
│       └── ts/               # Generated TypeScript code (language first, then service)
│           ├── package.json  # npm package: @example/contracts
│           ├── servicea/v1/
│           │   ├── servicea_pb.ts        # Generated protobuf types
│           │   └── servicea_connect.ts   # Generated Connect stubs
│           └── serviceb/v1/
│               ├── serviceb_pb.ts        # Generated
│               └── serviceb_connect.ts   # Generated
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

## Test Organization Summary

- **Unit Tests**: Co-located in `services/*/internal/domain/**/*_test.go` and `services/*/internal/application/**/*_test.go`
- **Integration Tests**: Co-located in `services/*/internal/adapters/**/*_test.go` (with real infrastructure)
- **Contract Tests**: Per-service in `services/*/test/contract/` (each service tests its OWN API contracts, HTTP responses, etc.)
- **E2E Tests**: Root-level `test/e2e/` (complete user journeys across all services)

**Important:** Contract tests live in the **PROVIDER** service, not the consumer. Example: `services/serviceasvc/test/contract/contracts_test.go` verifies that serviceasvc correctly implements `serviceasvc.ServiceAService`. The consumer (servicebsvc) never imports the provider's implementation.

See [testing-strategy.md](testing-strategy.md) for detailed testing guidance.
