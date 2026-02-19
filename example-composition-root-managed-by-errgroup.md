# Composition Root: Direct Explicit Wiring with errgroup Supervisor

The monolith's `main.go` performs direct explicit wiring with clear initialization order. This shows the **Pure Contract Definition Pattern** with truly independent modules.

## The Complete main.go

**File:** `cmd/monolith/main.go`

```go
package main

import (
    "context"
    "log"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"

    "golang.org/x/sync/errgroup"

    // Contract definition modules (public contracts)
    serviceasvc "github.com/example/service-manager/contracts/definitions/serviceasvc"

    // Service config packages
    serviceaconfig "github.com/example/service-manager/services/serviceasvc/config"
    servicebconfig "github.com/example/service-manager/services/servicebsvc/config"

    // Service adapters
    serviceaadapters "github.com/example/service-manager/services/serviceasvc/internal/adapters/inbound/contracts"
    serviceahttp "github.com/example/service-manager/services/serviceasvc/internal/adapters/inbound/http"

    servicebsvc "github.com/example/service-manager/services/servicebsvc"
    servicebhttp "github.com/example/service-manager/services/servicebsvc/internal/adapters/inbound/http"
)

func main() {
    // ============================================================
    // 1. Root Context with Signal Handling (SIGTERM/SIGINT)
    // ============================================================
    ctx, cancel := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
    defer cancel()

    // ============================================================
    // 2. Configuration Loading
    // ============================================================
    // Each service defines HOW to load its config (Embedded FS + Env Override)
    // main.go decides WHEN to load it

    authorCfg, err := serviceaconfig.Load()
    if err != nil {
        log.Fatalf("Failed to load author service config: %v", err)
    }

    authCfg, err := servicebconfig.Load()
    if err != nil {
        log.Fatalf("Failed to load auth service config: %v", err)
    }

    // ============================================================
    // 3. Wiring Phase (Direct Explicit Dependency Injection)
    // ============================================================

    // A. Initialize Provider Service (serviceasvc)
    //    Creates InprocServer in service internal adapters
    //    Returns: serviceasvc.ServiceAService interface
    authorServer := serviceaadapters.NewInprocServer(
        authorCfg.DB,
        authorCfg.Logger,
    )

    // Create HTTP handler for author service
    authorHTTPHandler := serviceahttp.NewHandler(authorCfg)

    // B. Wrap Provider with Contract Definition Client
    //    Lives in contract definition module, accepts ServiceAService interface
    authorClient := serviceasvc.NewInprocClient(authorServer)

    // C. Initialize Consumer Service (servicebsvc)
    //    Accepts the contract client as a dependency
    authService := servicebsvc.New(authCfg, authorClient)

    // Create HTTP handler for auth service
    authHTTPHandler := servicebhttp.NewHandler(authCfg, authService)

    // ============================================================
    // 4. Execution Phase (The Supervisor - Shared Fate)
    // ============================================================
    // Use errgroup for concurrent service management
    // If ANY service fails, ALL services shut down (Shared Fate)

    g, gCtx := errgroup.WithContext(ctx)

    // --- Start Service A HTTP Server ---
    g.Go(func() error {
        log.Printf("Starting Service A on %s", authorCfg.HTTPPort)
        server := &http.Server{
            Addr:    authorCfg.HTTPPort,
            Handler: authorHTTPHandler,
        }

        // Graceful shutdown watcher
        go func() {
            <-gCtx.Done()
            log.Println("Shutting down Service A...")
            shutdownCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
            defer cancel()
            if err := server.Shutdown(shutdownCtx); err != nil {
                log.Printf("Service A shutdown error: %v", err)
            }
        }()

        if err := server.ListenAndServe(); err != http.ErrServerClosed {
            return err // Error propagates to errgroup -> cancels gCtx -> stops all services
        }
        return nil
    })

    // --- Start Service B HTTP Server ---
    g.Go(func() error {
        log.Printf("Starting Service B on %s", authCfg.HTTPPort)
        server := &http.Server{
            Addr:    authCfg.HTTPPort,
            Handler: authHTTPHandler,
        }

        go func() {
            <-gCtx.Done()
            log.Println("Shutting down Service B...")
            shutdownCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
            defer cancel()
            if err := server.Shutdown(shutdownCtx); err != nil {
                log.Printf("Service B shutdown error: %v", err)
            }
        }()

        if err := server.ListenAndServe(); err != http.ErrServerClosed {
            return err
        }
        return nil
    })

    // ============================================================
    // 5. Blocking Wait (Shared Fate Architecture)
    // ============================================================
    // If ANY service returns an error, ALL services shut down
    // This ensures no "zombie" processes with half the services running

    if err := g.Wait(); err != nil {
        log.Fatalf("Monolith shutdown due to error: %v", err)
    }

    log.Println("Monolith shutdown gracefully")
}
```

---

## Configuration Pattern: Embedded FS + Env Override

Each service implements its own config loader following this pattern:

**File:** `services/serviceasvc/config/config.go`

```go
package config

import (
    "embed"
    "fmt"
    "os"

    "gopkg.in/yaml.v3"
)

//go:embed defaults.yaml
var defaultsFS embed.FS

type Config struct {
    HTTPPort string `yaml:"http_port"`
    DB       DBConfig `yaml:"db"`
    Logger   LoggerConfig `yaml:"logger"`
}

type DBConfig struct {
    Host     string `yaml:"host"`
    Port     int    `yaml:"port"`
    Database string `yaml:"database"`
    User     string `yaml:"user"`
    Password string `yaml:"password"`
}

type LoggerConfig struct {
    Level  string `yaml:"level"`
    Format string `yaml:"format"`
}

// Load reads embedded defaults and applies environment variable overrides
func Load() (*Config, error) {
    // 1. Load embedded defaults
    data, err := defaultsFS.ReadFile("defaults.yaml")
    if err != nil {
        return nil, fmt.Errorf("failed to read defaults: %w", err)
    }

    var cfg Config
    if err := yaml.Unmarshal(data, &cfg); err != nil {
        return nil, fmt.Errorf("failed to parse defaults: %w", err)
    }

    // 2. Apply environment variable overrides
    if port := os.Getenv("SERVICEASVC_HTTP_PORT"); port != "" {
        cfg.HTTPPort = port
    }
    if host := os.Getenv("SERVICEASVC_DB_HOST"); host != "" {
        cfg.DB.Host = host
    }
    if dbName := os.Getenv("SERVICEASVC_DB_NAME"); dbName != "" {
        cfg.DB.Database = dbName
    }
    if password := os.Getenv("SERVICEASVC_DB_PASSWORD"); password != "" {
        cfg.DB.Password = password
    }

    return &cfg, nil
}
```

**File:** `services/serviceasvc/config/defaults.yaml`

```yaml
http_port: ":8081"
db:
  host: "localhost"
  port: 5432
  database: "serviceasvc"
  user: "postgres"
  password: "postgres"
logger:
  level: "info"
  format: "json"
```

---

## Key Architectural Principles

### 1. Direct Explicit Wiring

**No registry pattern**, **no dependency injection framework** - just explicit initialization in main.go:

```go
authorServer := serviceaadapters.NewInprocServer(...)  // Step 1: Create server
authorClient := serviceasvc.NewInprocClient(authorServer)  // Step 2: Wrap in client
authService := servicebsvc.New(authCfg, authorClient)  // Step 3: Inject into consumer
```

**Benefits:**
- Initialization order is visible and explicit
- No hidden magic, no globals, no singletons
- Easy to understand and debug
- Compiler enforces type safety

### 2. Configuration Ownership

Each service owns its configuration:
- **Service defines HOW**: Implements `config.Load()` with embedded defaults + env overrides
- **main.go decides WHEN**: Calls `config.Load()` at the appropriate time
- **No global config**: Each service gets its own config object

**Benefits:**
- Services are self-contained (can run standalone)
- No coupling through shared config
- Easy to test (mock config.Load())

### 3. Interface-Based Coupling

Everything operates on interfaces, not concrete types:

```go
authorServer := serviceaadapters.NewInprocServer(...)  // Returns ServiceAService interface
authorClient := serviceasvc.NewInprocClient(authorServer)  // Accepts ServiceAService interface
```

**Benefits:**
- Loose coupling between provider and consumer
- Easy to swap implementations (in-process vs network)
- Testable (mock the interface)

### 4. Shared Fate Architecture (errgroup)

If ANY service fails, ALL services shut down:

```go
g.Go(func() error { return authorServer.Run(gCtx) })
g.Go(func() error { return authService.Run(gCtx) })

if err := g.Wait(); err != nil {
    log.Fatal(err)  // Entire monolith exits
}
```

**Why Shared Fate?**
- Prevents "zombie" processes (half the services running, half dead)
- Ensures clean state (either all healthy or all restarted)
- Leverages Kubernetes/Docker restart policies
- Simpler than partial failure handling

**Alternative:** For fine-grained fault tolerance (individual service restarts), consider [suture](https://github.com/thejerf/suture) for Erlang-style supervision trees.

### 5. Graceful Shutdown

Each service watches for context cancellation and shuts down gracefully:

```go
go func() {
    <-gCtx.Done()  // Context cancelled (SIGTERM or service failure)
    shutdownCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    server.Shutdown(shutdownCtx)  // Graceful shutdown with timeout
}()
```

**Benefits:**
- In-flight requests complete
- Database connections close cleanly
- No data corruption

---

## Wiring Flow Diagram

```mermaid
graph TD
    Main[main.go]

    Main -->|1. Load Config| AuthorCfg[serviceaconfig.Load]
    Main -->|2. Load Config| AuthCfg[servicebconfig.Load]

    Main -->|3. Create Server| AuthorServer[serviceaadapters.NewInprocServer]
    AuthorServer -->|returns| AuthorIface[ServiceAService interface]

    Main -->|4. Wrap in Client| AuthorClient[serviceasvc.NewInprocClient]
    AuthorIface -->|passed to| AuthorClient

    Main -->|5. Initialize Service| AuthService[servicebsvc.New]
    AuthorClient -->|injected into| AuthService

    Main -->|6. errgroup| Supervisor[g.Go for each service]

    style Main fill:#e1f5ff
    style AuthorIface fill:#fff4e1
    style Supervisor fill:#ffe1e1
```

---

## Verification

After implementing this pattern, verify:

- [ ] No registry struct or singleton pattern
- [ ] Initialization order is explicit in main.go
- [ ] Each service has its own `config.Load()` implementation
- [ ] InprocServer is in `services/*/internal/adapters/inbound/contracts/definitions/`
- [ ] InprocClient is in `contracts/definitions/*/`
- [ ] All types are interface-based (no `*InprocServer` concrete types)
- [ ] errgroup manages service lifecycle (Shared Fate)
- [ ] Graceful shutdown handlers for all HTTP servers
- [ ] No `import "services/*/internal"` from other services
