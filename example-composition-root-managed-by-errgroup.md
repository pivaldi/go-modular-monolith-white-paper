`cmd/sm/main.go`:

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

    // Import service modules
    "github.com/example/service-manager/services/authsvc"
    "github.com/example/service-manager/services/authorsvc"

    // Import bridges
    authbridge "github.com/example/service-manager/bridge/authsvc"
    authorbridge "github.com/example/service-manager/bridge/authorsvc"
)

func main() {
    // 1. Root Context with Signal Handling (SIGTERM/SIGINT)
    ctx, cancel := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
    defer cancel()

    // 2. Load Global Config
    cfg := loadConfig()

    // 3. Wiring Phase (Dependency Injection)
    // --------------------------------------

    // A. Initialize Provider (Author Service)
    // We initialize the internal logic and get back the public InprocServer
    authorContainer := authorsvc.NewContainer(ctx, cfg.AuthorDB)

    // B. Initialize Consumer (Auth Service)
    // We inject the Author Service's InprocServer directly into Auth Service
    // The Compiler guarantees type safety via the Bridge Interface
    authContainer := authsvc.NewContainer(
        ctx, 
        cfg.AuthDB,
        // The Bridge: Adapts the Server to a Client interface
        authorbridge.NewInprocClient(authorContainer.BridgeServer), 
    )

    // 4. Execution Phase (The Supervisor)
    // -----------------------------------
    g, gCtx := errgroup.WithContext(ctx)

    // --- Start Author Service HTTP Server ---
    g.Go(func() error {
        log.Printf("Starting Author Service on %s", cfg.AuthorPort)
        server := &http.Server{
            Addr:    cfg.AuthorPort,
            Handler: authorContainer.HTTPHandler,
        }

        // Graceful shutdown watcher
        go func() {
            <-gCtx.Done()
            shutdownCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
            defer cancel()
            server.Shutdown(shutdownCtx)
        }()

        if err := server.ListenAndServe(); err != http.ErrServerClosed {
            return err // Returns error to errgroup -> cancels gCtx -> stops other services
        }
        return nil
    })

    // --- Start Auth Service HTTP Server ---
    g.Go(func() error {
        log.Printf("Starting Auth Service on %s", cfg.AuthPort)
        server := &http.Server{
            Addr:    cfg.AuthPort,
            Handler: authContainer.HTTPHandler,
        }

        go func() {
            <-gCtx.Done()
            shutdownCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
            defer cancel()
            server.Shutdown(shutdownCtx)
        }()

        if err := server.ListenAndServe(); err != http.ErrServerClosed {
            return err
        }
        return nil
    })

    // 5. Blocking Wait
    // If ANY service returns an error, we shut down EVERYTHING.
    if err := g.Wait(); err != nil {
        log.Fatalf("Monolith shutdown due to error: %v", err)
    }

    log.Println("Monolith shutdown gracefully")
}
```

