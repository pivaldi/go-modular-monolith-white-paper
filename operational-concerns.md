# Operational Concerns

## CI/CD Pipeline

**Key stages:**

1. **Validate** - Check workspace, go.mod files
2. **Lint** - golangci-lint, buf lint
3. **Architecture Check** - Enforce boundaries
4. **Unit Tests** - Fast tests without infrastructure
5. **Integration Tests** - With databases
6. **Contract Tests** - Service-to-service
7. **Build** - Docker images
8. **Security Scan** - Gosec, Trivy

**Example GitHub Actions:**

```yaml
name: CI
on: [push, pull_request]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.22'
      - run: go work sync
      - run: go mod verify

  arch-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
      - run: go run ./tools/arch-test

  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: postgres
        ports:
          - 5432:5432
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
      - run: go test ./... -race -coverprofile=coverage.out
```

## Architectural Boundary Enforcement

Architecture tests validate that code structure matches intended design. These tests should run in CI and fail the build on violations.

### Architecture Testing Tools

**1. Custom Architecture Tests (`tools/arch-test/main.go`)**

A comprehensive validation tool that checks:

**Service Isolation:**
- Services don't import other service `internal/` packages
- Services only communicate via bridge modules or network
- No cross-service imports except through public contracts

**Layer Purity:**
- Domain layer has zero infrastructure dependencies
- Domain doesn't import `database/sql`, `net/http`, or framework packages
- Application layer doesn't import adapters
- Adapters implement application ports (dependency inversion)

**Module Dependencies:**
- Bridge modules have no dependencies (pure interfaces)
- Services depend only on: bridges, contracts, and standard libraries
- No circular dependencies between modules (at go.mod level - compiler already prevents package-level cycles)
- Dependency graph flows in correct direction

**Import Graph Validation:**
- Validate module dependency graph
- Detect transitive dependency violations
- Ensure clean dependency tree
- Use companion visualization tools (see below)

**Complete Implementation:**

See the file [arch-test.md](arch-test.md)

### 2. Third-Party Tools

**Architecture Testing:**
- [arch-go](https://github.com/fdaines/arch-go): Comprehensive architecture testing
- [go-cleanarch](https://github.com/roblaszczak/go-cleanarch): Layer dependency validation

**Dependency Visualization:**

Use these companion tools to visualize and analyze your dependency graph:

- **[godepgraph](https://github.com/kisielk/godepgraph)** (Recommended)
  - Generates dependency graphs in Graphviz DOT format
  - Color-codes different package types
  - Simple, focused tool for package-level dependencies

  ```bash
  # Install
  go install github.com/kisielk/godepgraph@latest

  # Generate visualization
  godepgraph -s github.com/yourorg/yourproject | dot -Tpng -o deps.png

  # Focus on specific package
  godepgraph -s -o github.com/yourorg/yourproject/services/authsvc | dot -Tpng -o authsvc-deps.png
  ```

- **[goda](https://github.com/loov/goda)** (Advanced)
  - Comprehensive dependency analysis toolkit
  - Multiple visualization commands (graph, tree, list)
  - Supports filtering and clustering
  - Can output directly to SVG

  ```bash
  # Install
  go install github.com/loov/goda@latest

  # Generate graph with clustering
  goda graph "./..." | dot -Tsvg -o deps.svg

  # Show dependency tree
  goda tree "./..."

  # Analyze specific service dependencies
  goda graph "reach(./services/authsvc/...)" | dot -Tpng -o authsvc-reach.png
  ```

- **[graphdot](https://github.com/ewohltman/graphdot)** (Module-level)
  - Visualizes Go module dependencies (go.mod level)
  - Good for understanding module-level architecture
  - Run in project root to generate DOT output

  ```bash
  # Install
  go install github.com/ewohltman/graphdot@latest

  # Generate module dependency graph
  graphdot | dot -Tpng -o modules.png
  ```

**Usage Recommendations:**

1. **During Development**: Use `godepgraph` to quickly check service boundaries
2. **Architecture Review**: Use `goda` for comprehensive dependency analysis
3. **Module Planning**: Use `graphdot` to visualize module-level dependencies
4. **Documentation**: Include generated graphs in architecture documentation

### Integration with CI

Architecture tests should be:
- **Required** - Block merge if tests fail
- **Fast** - Run on every PR (~5-10 seconds)
- **Clear** - Provide actionable error messages
- **Comprehensive** - Cover all architectural rules

Example CI failure message:
```
✗ Architecture validation failed

Checking: service-isolation
  ✗ FAILED: services/authsvc/internal/adapters/outbound/helper.go
    imports services/authorsvc/internal/domain/author
    (cross-service internal import)

Fix: Remove direct import of authorsvc internals.
      Use bridge/authorsvc instead.
```

### Additional Validation Rules

**Naming Conventions:**
```go
// Ensure domain entities don't have infrastructure suffixes
func checkNamingConventions() error {
    forbiddenSuffixes := []string{"DTO", "Handler", "Controller", "Repository"}
    // Check domain layer files...
}
```

**Dependency Limits:**
```go
// Ensure services don't exceed dependency count
func checkDependencyCount() error {
    maxDependencies := 10
    // Count imports per file...
}
```

**Test Coverage:**
```go
// Ensure domain layer has high coverage
func checkDomainCoverage() error {
    minCoverage := 80.0 // percent
    // Parse coverage.out...
}
```

### References

- [go-cleanarch](https://github.com/bxcodec/go-clean-arch) - Clean Architecture reference implementation in Go
- [arch-go](https://github.com/fdaines/arch-go) - Architecture testing tool
- [golang-standards/project-layout](https://github.com/golang-standards/project-layout) - Standard Go project layout

**Note:** While go-cleanarch provides excellent examples of clean architecture in Go, our pattern adds:
- Go workspaces for true module isolation
- Bridge modules for explicit service boundaries
- Support for both in-process and network communication

## Observability

**Metrics (Prometheus):**

```go
// services/authsvc/internal/infra/observability/metrics.go
var (
    HTTPRequestsTotal = promauto.NewCounterVec(...)
    LoginAttemptsTotal = promauto.NewCounterVec(...)
    ActiveSessionsGauge = promauto.NewGauge(...)
    DatabaseQueryDuration = promauto.NewHistogramVec(...)
)
```

**Logging (Structured):**

```go
logger.Info(ctx, "user logged in",
    ports.LogField{Key: "user_id", Value: userID},
    ports.LogField{Key: "trace_id", Value: traceID},
)
```

**Tracing (OpenTelemetry):**

```go
tracer := otel.Tracer("authsvc")
ctx, span := tracer.Start(ctx, "LoginCommand.Execute")
defer span.End()
```

## Deployment

**Development:**
- All services run on localhost (in-process bridges)
- Single `mise run dev` command
- Hot reload via [air](https://github.com/air-verse/air).

**Production Options:**

**Option 1: Single deployment (all services in one process)**
```bash
# Build single binary that runs all services
go build -o bin/service-manager ./cmd/sm
```

**Option 2: Separate deployments (distributed)**
```bash
# Build separate binaries
go build -o bin/authsvc ./services/authsvc/cmd/authsvc
go build -o bin/authorsvc ./services/authorsvc/cmd/authorsvc

# Deploy to separate pods/hosts
# Services use Connect/HTTP to communicate
```

**Option 3: Hybrid (mix of co-located and distributed)**

- Co-locate non-critical services
- Distribute performance-critical services

