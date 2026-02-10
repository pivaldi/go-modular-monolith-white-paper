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
│   │   └── internal/         # PRIVATE - cannot be imported by other services
│   │       │
│   │       ├── domain/       # DOMAIN LAYER (pure business logic)
│   │       │   │             # No external dependencies
│   │       │   │             # No adapters, no infrastructure
│   │       │   │
│   │       │   ├── user/     # User aggregate
│   │       │   │   ├── user.go           # Entity (aggregate root)
│   │       │   │   ├── email.go          # Value object
│   │       │   │   ├── password.go       # Value object
│   │       │   │   └── repository.go     # Repository interface (port)
│   │       │   │
│   │       │   ├── session/  # Session aggregate
│   │       │   │   ├── session.go        # Entity
│   │       │   │   ├── token.go          # Value object
│   │       │   │   └── repository.go     # Repository interface
│   │       │   │
│   │       │   └── errors.go # Domain errors
│   │       │
│   │       ├── application/  # APPLICATION LAYER (use cases)
│   │       │   │             # Orchestrates domain objects
│   │       │   │             # Depends on: domain, ports (interfaces only)
│   │       │   │
│   │       │   ├── command/  # Write operations
│   │       │   │   ├── login.go          # Login use case
│   │       │   │   ├── logout.go         # Logout use case
│   │       │   │   ├── register.go       # Registration use case
│   │       │   │   └── change_password.go
│   │       │   │
│   │       │   ├── query/    # Read operations
│   │       │   │   ├── get_user.go
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
│   │       │   │             # Implements ports from application layer
│   │       │   │
│   │       │   ├── inbound/  # Inbound adapters (primary/driving)
│   │       │   │   │         # External world -> Application
│   │       │   │   │
│   │       │   │   ├── http/ # HTTP REST adapter
│   │       │   │   │   ├── server.go
│   │       │   │   │   ├── handlers/
│   │       │   │   │   │   ├── login.go
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
│   │       │       │   │   ├── session_repository.go
│   │       │       │   │   └── migrations/
│   │       │       │   │
│   │       │       │   └── memory/   # In-memory (testing)
│   │       │       │
│   │       │       ├── authorclient/  # Author service client adapters
│   │       │       │   ├── inproc/   # In-process adapter (TODAY)
│   │       │       │   │   └── client.go
│   │       │       │   │       # Implements ports.AuthorClient
│   │       │       │   │       # Uses bridge/authorsvc.InprocClient
│   │       │       │   │       # Zero network overhead
│   │       │       │   │
│   │       │       │   └── connect/  # Network adapter (LATER)
│   │       │       │       └── client.go
│   │       │       │           # Implements ports.AuthorClient
│   │       │       │           # Uses authorconnect.Client
│   │       │       │           # Network overhead (HTTP, serialization)
│   │       │       │
│   │       │       ├── cache/
│   │       │       │   ├── redis/
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
│       └── internal/
│           ├── domain/
│           │   └── author/   # Author aggregate
│           │       ├── author.go
│           │       ├── author_id.go
│           │       ├── name.go
│           │       └── repository.go
│           │
│           ├── application/
│           │   ├── command/
│           │   │   ├── create_author.go
│           │   │   └── update_author.go
│           │   │
│           │   ├── query/
│           │   │   ├── get_author.go
│           │   │   └── list_authors.go
│           │   │
│           │   └── ports/
│           │       ├── image_client.go
│           │       └── logger.go
│           │
│           ├── adapters/
│           │   ├── inbound/      # Inbound adapters (primary/driving)
│           │   │   │
│           │   │   ├── bridge/   # Bridge adapter (in-process)
│           │   │   │   └── inproc_server.go
│           │   │   │       # Implements bridge.AuthorService interface
│           │   │   │       # Wraps authorsvc/internal/application
│           │   │   │       # CAN import authorsvc/internal (same module!)
│           │   │   │       # Returns AuthorService interface for loose coupling
│           │   │   │
│           │   │   ├── http/     # HTTP REST adapter
│           │   │   │   └── ...
│           │   │   │
│           │   │   └── connect/  # Connect/gRPC adapter
│           │   │       └── ...
│           │   │
│           │   └── outbound/
│           │       ├── persistence/postgres/
│           │       ├── imageclient/
│           │       └── cache/
│           │
│           └── infra/
│
├── tools/
│   └── arch-test/            # Architectural boundary enforcement
│       └── main.go           # Validates import rules in CI
│
└── docs/
    ├── architecture/
    │   ├── decisions/        # Architecture Decision Records
    │   └── diagrams/
    │
    └── development/
        ├── local-setup.md
        ├── testing-guide.md
        └── adding-a-service.md
```
