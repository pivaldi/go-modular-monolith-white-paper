# Operational Concerns

## CI/CD Pipeline

**Key stages:**

1. **Validate** - Check workspace, go.mod files
2. **Lint** - golangci-lint, buf lint
3. **Architecture Check** - Enforce boundaries via `mmw check arch`
4. **Unit Tests** - Fast tests without infrastructure
5. **Integration Tests** - With databases
6. **Contract Tests** - Module-to-module
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
          go-version: '1.23'
      - uses: bufbuild/buf-setup-action@v1
      - run: go work sync
        working-directory: poc
      - run: go mod verify
        working-directory: poc
      - run: buf lint
        working-directory: poc/contracts
      - run: buf generate --template buf.gen.todo.yaml
        working-directory: poc/contracts
      - run: buf generate --template buf.gen.auth.yaml
        working-directory: poc/contracts
      - run: git diff --exit-code
        name: Verify generated code is up to date

  arch-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
      - run: go run ./cmd/mmw-cli check arch
        working-directory: poc/mmw

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
        working-directory: poc
```

## Architectural Boundary Enforcement

Architecture tests validate that code structure matches intended design.
These tests run via `mmw check arch` and should fail the build on violations.

### Architecture Testing with `mmw check arch`

The `mmw check arch` CLI command (from `mmw/cmd/mmw-cli/`) runs all registered
`Validator` implementations against the workspace. Built-in validators check:

**Module Isolation:**
- Modules don't import other modules' `internal/` packages
- Modules only communicate via contract packages or the network
- No cross-module imports except through `contracts/go/application/{domain}/`

**Layer Purity:**
- Domain layer has zero infrastructure dependencies
- Domain doesn't import `database/sql`, `net/http`, or framework packages
- Application layer doesn't import adapters
- Adapters implement application ports (dependency inversion)

**Contract Module Purity:**
- Contract packages depend only on `connectrpc/connect` and `google.golang.org/protobuf`
- No feature module can be imported by a contract package (structurally impossible
  — would create a circular module dependency)

See [arch-test.md](arch-test.md) for the full implementation reference.

### Third-Party Tools

**Architecture Testing:**
- [arch-go](https://github.com/fdaines/arch-go): Comprehensive architecture testing
- [go-cleanarch](https://github.com/roblaszczak/go-cleanarch): Layer dependency validation

**Dependency Visualization:**

Use these companion tools to visualize and analyze your dependency graph:

- **[godepgraph](https://github.com/kisielk/godepgraph)** (Recommended)
  - Generates dependency graphs in Graphviz DOT format
  - Simple, focused tool for package-level dependencies

  ```bash
  go install github.com/kisielk/godepgraph@latest
  godepgraph -s github.com/pivaldi/mmw-todo | dot -Tpng -o todo-deps.png
  ```

- **[goda](https://github.com/loov/goda)** (Advanced)
  - Comprehensive dependency analysis toolkit

  ```bash
  go install github.com/loov/goda@latest
  goda graph "reach(./modules/todo/...)" | dot -Tpng -o todo-reach.png
  ```

- **[graphdot](https://github.com/ewohltman/graphdot)** (Module-level)
  - Visualizes Go module dependencies (go.mod level)

  ```bash
  go install github.com/ewohltman/graphdot@latest
  graphdot | dot -Tpng -o modules.png
  ```

### Integration with CI

Architecture tests should be:
- **Required** — Block merge if tests fail
- **Fast** — Run on every PR (~5-10 seconds)
- **Clear** — Provide actionable error messages
- **Comprehensive** — Cover all architectural rules

Example CI failure message:
```
✗ Architecture validation failed

Checking: module-isolation
  ✗ FAILED: modules/todo/internal/adapters/outbound/helper.go
    imports modules/auth/internal/domain/user
    (cross-module internal import)

Fix: Remove direct import of auth internals.
      Use contracts/go/application/auth instead.
```

## Observability

**Structured Logging:**

```go
// modules/todo/internal/adapters/inbound/connect/todo_handler.go
slog.InfoContext(ctx, "todo created",
    slog.String("todo_id", todo.ID().String()),
    slog.String("user_id", userID.String()),
)
```

## Deployment

**Development:**
- All modules run in-process (contract interfaces satisfied by direct method calls)
- Single `mise run dev` command
- Hot reload via [air](https://github.com/air-verse/air)

**Production Options:**

**Option 1: Single deployment (all modules in one process)**
```bash
go build -o bin/mmw ./cmd/mmw
./bin/mmw
```

**Option 2: Separate deployments (distributed)**

Replace in-process accessors with HTTP clients in the composition root:
```go
// cmd/mmw/main.go — after extracting auth to its own pod
todoModule, _ := todo.New(todo.Infrastructure{
    AuthSvc: defauth.NewPrivateHTTPClient(
        authv1connect.NewAuthPrivateServiceClient(&http.Client{}, "https://auth.internal"),
    ),
})
```

No application code changes — only the composition root changes.

**Option 3: Hybrid (mix of co-located and distributed)**

- Co-locate modules with shared transactional boundaries
- Distribute modules with independent scaling requirements
