# Complete Directory Structure

```
service-manager/
├── go.work                    # Go workspace definition
│                              # use (./contracts ./bridge/authorsvc ./services/...)
│
├── mise.toml                  # Root orchestration tasks
│
├── buf.yaml                   # Buf workspace config (optional)
├── buf.gen.yaml               # Protobuf code generation (optional)
│
├── contracts/                # Protobuf definitions (OPTIONAL - for network transport)
│   ├── go.mod                # Module: contracts v1.0.0
│   ├── README.md             # Contract versioning strategy
│   │
│   ├── auth/v1/
│   │   ├── auth.proto        # AuthService definition
│   │   ├── auth.pb.go        # Generated protobuf types
│   │   └── authconnect/
│   │       └── auth.connect.go # Generated Connect stubs
│   │
│   └── author/v1/
│       ├── author.proto      # AuthorService definition
│       ├── author.pb.go      # Generated
│       └── authorconnect/
│           └── author.connect.go # Generated
│
├── bridge/                    # Bridge modules (public service APIs)
│   │
│   └── authorsvc/            # Author service public API
│       ├── go.mod            # Module: bridge/authorsvc v1.0.0
│       │                     # Dependencies: ZERO (truly dependency-free)
│       │
│       ├── README.md         # Bridge usage documentation
│       │
│       ├── api.go            # Public service interface
│       │   # type AuthorService interface {
│       │   #   GetAuthor(ctx, id) (*AuthorDTO, error)
│       │   #   CreateAuthor(ctx, req) (*AuthorDTO, error)
│       │   # }
│       │
│       ├── dto.go            # Domain-friendly DTOs
│       │   # type AuthorDTO struct { ID, Name, Bio string }
│       │
│       ├── errors.go         # Public error types
│       │   # var ErrAuthorNotFound = errors.New("author not found")
│       │
│       └── inproc_client.go  # In-process client (thin wrapper)
│           # Implements AuthorService interface
│           # Accepts AuthorService interface (not concrete type!)
│           # Calls via interface (function call, no network)
│
├── services/
│   │
│   ├── authsvc/              # Authentication Service
│   │   ├── go.mod            # Module: services/authsvc v2.1.0
│   │   │                     # Dependencies: bridge/authorsvc, contracts (optional)
│   │   │
│   │   ├── README.md         # Service documentation
│   │   ├── mise.toml         # Service-specific tasks
│   │   ├── Dockerfile
│   │   │
│   │   ├── cmd/
│   │   │   └── authsvc/
│   │   │       └── main.go   # Composition root
│   │   │           # Wires dependencies:
│   │   │           # - In dev: uses bridge.InprocClient
│   │   │           # - In prod: uses authorconnect.Client
│   │   │
│   │   ├── test/             # Service-level tests
│   │   │   └── contract/     # CONTRACT TESTS (verify authsvc's own API contracts)
│   │   │       └── http_api_test.go  # Tests authsvc HTTP responses match spec
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
│   │       │       ├── author_client.go  # Outbound port
│   │       │       │   # type AuthorClient interface {
│   │       │       │   #   GetAuthor(ctx, id) (*AuthorInfo, error)
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
│   │       │   │           └── auth_handler.go
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
│   │       │       ├── authorclient/  # Author service client adapters
│   │       │       │   ├── inproc/   # In-process adapter (TODAY)
│   │       │       │   │   ├── client.go
│   │       │       │   │   │   # Implements ports.AuthorClient
│   │       │       │   │   │   # Uses bridge/authorsvc.InprocClient
│   │       │       │   │   │   # Zero network overhead
│   │       │       │   │   └── client_test.go     # INTEGRATION TEST
│   │       │       │   │
│   │       │       │   └── connect/  # Network adapter (LATER)
│   │       │       │       ├── client.go
│   │       │       │       │   # Implements ports.AuthorClient
│   │       │       │       │   # Uses authorconnect.Client
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
│   └── authorsvc/            # Author Service (similar structure)
│       ├── go.mod            # Module: services/authorsvc v1.0.0
│       │                     # Dependencies: bridge/authorsvc (to implement server)
│       │
│       ├── cmd/authorsvc/
│       │   └── main.go       # Wires bridge.InprocServer to internal/application
│       │
│       ├── test/             # Service-level tests
│       │   └── contract/     # CONTRACT TESTS (verify authorsvc implements bridge correctly)
│       │       ├── bridge_test.go         # Tests InprocServer implements bridge.AuthorService
│       │       └── http_api_test.go       # Tests HTTP API matches spec
│       │
│       └── internal/
│           ├── domain/       # UNIT TESTS (co-located *_test.go)
│           │   └── author/   # Author aggregate
│           │       ├── author.go
│           │       ├── author_test.go    # UNIT TEST
│           │       ├── author_id.go
│           │       ├── name.go
│           │       ├── name_test.go      # UNIT TEST
│           │       └── repository.go
│           │
│           ├── application/  # UNIT TESTS (co-located *_test.go with mocked ports)
│           │   ├── command/
│           │   │   ├── create_author.go
│           │   │   ├── create_author_test.go  # UNIT TEST (mocked ports)
│           │   │   └── update_author.go
│           │   │
│           │   ├── query/
│           │   │   ├── get_author.go
│           │   │   ├── get_author_test.go     # UNIT TEST (mocked ports)
│           │   │   └── list_authors.go
│           │   │
│           │   └── ports/
│           │       ├── image_client.go
│           │       └── logger.go
│           │
│           ├── adapters/     # INTEGRATION TESTS (co-located *_test.go with real infrastructure)
│           │   ├── inbound/      # Inbound adapters (primary/driving)
│           │   │   │
│           │   │   ├── bridge/   # Bridge adapter (in-process)
│           │   │   │   ├── inproc_server.go
│           │   │   │   │   # Implements bridge.AuthorService interface
│           │   │   │   │   # Wraps authorsvc/internal/application
│           │   │   │   │   # CAN import authorsvc/internal (same module!)
│           │   │   │   │   # Returns AuthorService interface for loose coupling
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
│           │       │   ├── author_repository.go
│           │       │   └── author_repository_test.go  # INTEGRATION TEST (real DB)
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
- **Contract Tests**: Per-service in `services/*/test/contract/` (each service tests its OWN API contracts - bridge implementation, HTTP responses, etc.)
- **E2E Tests**: Root-level `test/e2e/` (complete user journeys across all services)

**Important:** Contract tests live in the **PROVIDER** service, not the consumer. Example: `services/authorsvc/test/contract/bridge_test.go` verifies that authorsvc correctly implements `bridge/authorsvc.AuthorService`. The consumer (authsvc) never imports the provider's implementation.

See [testing-strategy.md](testing-strategy.md) for detailed testing guidance.
