# Architecture Testing

## Complete Implementation

```go
// tools/arch-test/main.go
package main

import (
    "fmt"
    "go/parser"
    "go/token"
    "os"
    "path/filepath"
    "strings"
)

type ArchRule struct {
    Name        string
    Description string
    Check       func() error
}

func main() {
    rules := []ArchRule{
        {
            Name:        "service-isolation",
            Description: "Services must not import other services' internal packages",
            Check:       checkServiceIsolation,
        },
        {
            Name:        "domain-purity",
            Description: "Domain layer must not import infrastructure",
            Check:       checkDomainPurity,
        },
        {
            Name:        "adapter-direction",
            Description: "Adapters depend on ports, not vice versa",
            Check:       checkAdapterDirection,
        },
        {
            Name:        "bridge-purity",
            Description: "Bridge modules must have zero dependencies",
            Check:       checkBridgePurity,
        },
        {
            Name:        "module-dependencies",
            Description: "Validate module dependency graph (checks go.mod, not package imports)",
            Check:       checkModuleDependencies,
        },
        {
            Name:        "layer-dependencies",
            Description: "Layers must depend only on inner layers",
            Check:       checkLayerDependencies,
        },
    }

    fmt.Println("Validating architectural boundaries...\n")

    failed := false
    for _, rule := range rules {
        fmt.Printf("Checking: %s\n", rule.Name)
        fmt.Printf("  %s\n", rule.Description)

        if err := rule.Check(); err != nil {
            fmt.Printf("  ✗ FAILED: %v\n\n", err)
            failed = true
        } else {
            fmt.Printf("  ✓ PASSED\n\n")
        }
    }

    if failed {
        fmt.Println("✗ Architecture validation failed")
        os.Exit(1)
    }

    fmt.Println("✓ All architecture checks passed")
}

// checkServiceIsolation ensures services don't import each other's internals
func checkServiceIsolation() error {
    servicesDir := "services"
    services, err := os.ReadDir(servicesDir)
    if err != nil {
        return err
    }

    for _, service := range services {
        if !service.IsDir() {
            continue
        }

        servicePath := filepath.Join(servicesDir, service.Name())

        err := filepath.Walk(servicePath, func(path string, info os.FileInfo, err error) error {
            if err != nil {
                return err
            }

            if !strings.HasSuffix(path, ".go") || strings.HasSuffix(path, "_test.go") {
                return nil
            }

            fset := token.NewFileSet()
            f, err := parser.ParseFile(fset, path, nil, parser.ImportsOnly)
            if err != nil {
                return err
            }

            for _, imp := range f.Imports {
                importPath := strings.Trim(imp.Path.Value, "\"")

                // Check if importing another service's internal
                for _, otherService := range services {
                    if otherService.Name() == service.Name() {
                        continue
                    }

                    forbiddenImport := fmt.Sprintf("services/%s/internal", otherService.Name())
                    if strings.Contains(importPath, forbiddenImport) {
                        return fmt.Errorf(
                            "%s imports %s (cross-service internal import)",
                            path, importPath,
                        )
                    }
                }
            }

            return nil
        })

        if err != nil {
            return err
        }
    }

    return nil
}

// checkDomainPurity ensures domain layer has no infrastructure dependencies
func checkDomainPurity() error {
    forbiddenImports := []string{
        "database/sql",
        "net/http",
        "/adapters/",
        "/infra/",
        "github.com/gin-gonic",
        "github.com/gorilla",
        "gorm.io",
        "connectrpc.com",
    }

    return filepath.Walk("services", func(path string, info os.FileInfo, err error) error {
        if err != nil {
            return err
        }

        // Only check domain layer files
        if !strings.Contains(path, "/internal/domain/") || !strings.HasSuffix(path, ".go") {
            return nil
        }

        fset := token.NewFileSet()
        f, err := parser.ParseFile(fset, path, nil, parser.ImportsOnly)
        if err != nil {
            return err
        }

        for _, imp := range f.Imports {
            importPath := strings.Trim(imp.Path.Value, "\"")

            for _, forbidden := range forbiddenImports {
                if strings.Contains(importPath, forbidden) {
                    return fmt.Errorf(
                        "%s: domain imports forbidden package %s",
                        path, importPath,
                    )
                }
            }
        }

        return nil
    })
}

// checkAdapterDirection ensures adapters implement ports, not vice versa
func checkAdapterDirection() error {
    return filepath.Walk("services", func(path string, info os.FileInfo, err error) error {
        if err != nil {
            return err
        }

        // Check application/ports files
        if !strings.Contains(path, "/application/ports/") || !strings.HasSuffix(path, ".go") {
            return nil
        }

        fset := token.NewFileSet()
        f, err := parser.ParseFile(fset, path, nil, parser.ImportsOnly)
        if err != nil {
            return err
        }

        // Ports must not import adapters
        for _, imp := range f.Imports {
            importPath := strings.Trim(imp.Path.Value, "\"")

            if strings.Contains(importPath, "/adapters/") {
                return fmt.Errorf(
                    "%s: port imports adapter (dependency inversion violated)",
                    path,
                )
            }
        }

        return nil
    })
}

// checkBridgePurity ensures bridge modules:
// 1. Have no external dependencies in go.mod (literally zero require statements)
// 2. Never import any internal/ packages (from any service)
func checkBridgePurity() error {
    bridgeModules, err := os.ReadDir("bridge")
    if err != nil {
        // Bridge directory might not exist yet
        return nil
    }

    for _, bridge := range bridgeModules {
        if !bridge.IsDir() {
            continue
        }

        bridgePath := filepath.Join("bridge", bridge.Name())

        // Check 1: Verify go.mod has NO external dependencies
        if err := checkGoModPurity(bridgePath, bridge.Name()); err != nil {
            return err
        }

        // Check 2: Verify bridge files never import internal/ packages
        if err := checkNoInternalImports(bridgePath, bridge.Name()); err != nil {
            return err
        }
    }

    return nil
}

// checkGoModPurity verifies a bridge module has zero external dependencies
func checkGoModPurity(bridgePath, bridgeName string) error {
    goModPath := filepath.Join(bridgePath, "go.mod")
    content, err := os.ReadFile(goModPath)
    if err != nil {
        return nil // go.mod might not exist yet
    }

    // Parse go.mod for require statements
    lines := strings.Split(string(content), "\n")
    inRequire := false
    for _, line := range lines {
        line = strings.TrimSpace(line)

        if strings.HasPrefix(line, "require (") {
            inRequire = true
            continue
        }

        if inRequire && line == ")" {
            inRequire = false
            continue
        }

        if inRequire || strings.HasPrefix(line, "require ") {
            // Allow only indirect dependencies (from go.sum)
            if !strings.Contains(line, "// indirect") && !strings.HasPrefix(line, "//") {
                // Check if it's an external dependency (contains '.')
                parts := strings.Fields(line)
                if len(parts) > 0 && strings.Contains(parts[0], ".") {
                    return fmt.Errorf(
                        "bridge/%s has external dependency: %s\n"+
                        "\n"+
                        "Bridges must have ZERO dependencies (no require statements).\n"+
                        "\n"+
                        "If you see service internals here, InprocServer is in the wrong location.\n"+
                        "Move InprocServer to: services/%s/internal/adapters/inbound/bridge/\n"+
                        "\n"+
                        "Bridge modules should contain ONLY:\n"+
                        "  - Interfaces (api.go)\n"+
                        "  - DTOs (dto.go)\n"+
                        "  - Errors (errors.go)\n"+
                        "  - InprocClient (thin wrapper)\n",
                        bridgeName, line, bridgeName,
                    )
                }
            }
        }
    }

    return nil
}

// checkNoInternalImports verifies bridge code never imports internal/ packages
func checkNoInternalImports(bridgePath, bridgeName string) error {
    err := filepath.Walk(bridgePath, func(path string, info os.FileInfo, err error) error {
        if err != nil {
            return err
        }

        if !strings.HasSuffix(path, ".go") || strings.HasSuffix(path, "_test.go") {
            return nil
        }

        fset := token.NewFileSet()
        f, err := parser.ParseFile(fset, path, nil, parser.ImportsOnly)
        if err != nil {
            return err
        }

        for _, imp := range f.Imports {
            importPath := strings.Trim(imp.Path.Value, "\"")

            // Bridge modules CANNOT import ANY internal/ packages
            if strings.Contains(importPath, "/internal/") {
                return fmt.Errorf(
                    "bridge/%s/%s: imports internal package: %s\n"+
                    "\n"+
                    "Bridge modules must NEVER import internal/ packages from any service.\n"+
                    "\n"+
                    "If this is InprocServer, it belongs in the service's internal adapters:\n"+
                    "  Move to: services/%s/internal/adapters/inbound/bridge/inproc_server.go\n"+
                    "\n"+
                    "Bridge modules can only import:\n"+
                    "  - Standard library\n"+
                    "  - Other bridge modules (public APIs)\n",
                    bridgeName, filepath.Base(path), importPath, bridgeName,
                )
            }
        }

        return nil
    })

    return err
}

// checkModuleDependencies validates the module dependency graph.
// NOTE: This checks MODULE-level dependencies (go.mod require statements),
// NOT package-level imports (which the Go compiler already validates).
// Go workspaces allow module cycles, but we forbid them architecturally.
func checkModuleDependencies() error {
    // Build dependency graph from go.mod files
    graph := make(map[string][]string)

    err := filepath.Walk(".", func(path string, info os.FileInfo, err error) error {
        if err != nil {
            return err
        }

        if info.Name() != "go.mod" {
            return nil
        }

        content, err := os.ReadFile(path)
        if err != nil {
            return err
        }

        // Extract module name
        lines := strings.Split(string(content), "\n")
        var moduleName string
        for _, line := range lines {
            if strings.HasPrefix(line, "module ") {
                moduleName = strings.TrimSpace(strings.TrimPrefix(line, "module "))
                break
            }
        }

        // Extract dependencies
        var deps []string
        inRequire := false
        for _, line := range lines {
            line = strings.TrimSpace(line)

            if strings.HasPrefix(line, "require (") {
                inRequire = true
                continue
            }

            if inRequire && line == ")" {
                inRequire = false
                continue
            }

            if inRequire || strings.HasPrefix(line, "require ") {
                parts := strings.Fields(line)
                if len(parts) > 0 && strings.Contains(parts[0], "service-manager") {
                    deps = append(deps, parts[0])
                }
            }
        }

        graph[moduleName] = deps
        return nil
    })

    if err != nil {
        return err
    }

    // Check for circular dependencies at MODULE level
    // NOTE: Go compiler prevents package-level cycles automatically.
    // However, Go workspaces ALLOW module-level cycles (A requires B, B requires A).
    // We forbid module cycles because:
    // - Bridge modules should be dependency-free (pure interfaces)
    // - Services should depend on bridges, not vice versa
    // - Circular module deps prevent independent evolution
    //
    // Example violation:
    //   services/servicebsvc/go.mod: require bridge/serviceasvc ✓ OK
    //   bridge/serviceasvc/go.mod: require services/servicebsvc ✗ CYCLE!
    for module := range graph {
        visited := make(map[string]bool)
        if hasCycle(module, graph, visited, make(map[string]bool)) {
            return fmt.Errorf("circular dependency detected involving %s", module)
        }
    }

    return nil
}

// hasCycle detects circular dependencies at the MODULE level using DFS.
// This checks go.mod dependencies, NOT package imports (compiler already prevents those).
func hasCycle(module string, graph map[string][]string, visited, recStack map[string]bool) bool {
    visited[module] = true
    recStack[module] = true

    for _, dep := range graph[module] {
        if !visited[dep] {
            if hasCycle(dep, graph, visited, recStack) {
                return true
            }
        } else if recStack[dep] {
            return true
        }
    }

    recStack[module] = false
    return false
}

// checkLayerDependencies ensures layers depend only on inner layers
func checkLayerDependencies() error {
    // Layer hierarchy (outer -> inner)
    // Infra can import: adapters, application, domain
    // Adapters can import: application, domain
    // Application can import: domain
    // Domain can import: nothing (except stdlib)

    layerRules := map[string][]string{
        "/internal/domain/":      {},
        "/internal/application/": {"/internal/domain/"},
        "/internal/adapters/":    {"/internal/application/", "/internal/domain/"},
        "/internal/infra/":       {"/internal/adapters/", "/internal/application/", "/internal/domain/"},
    }

    return filepath.Walk("services", func(path string, info os.FileInfo, err error) error {
        if err != nil {
            return err
        }

        if !strings.HasSuffix(path, ".go") || strings.HasSuffix(path, "_test.go") {
            return nil
        }

        // Determine current layer
        var currentLayer string
        for layer := range layerRules {
            if strings.Contains(path, layer) {
                currentLayer = layer
                break
            }
        }

        if currentLayer == "" {
            return nil
        }

        fset := token.NewFileSet()
        f, err := parser.ParseFile(fset, path, nil, parser.ImportsOnly)
        if err != nil {
            return err
        }

        allowedLayers := layerRules[currentLayer]

        for _, imp := range f.Imports {
            importPath := strings.Trim(imp.Path.Value, "\"")

            // Check if importing from a layer
            for layer := range layerRules {
                if !strings.Contains(importPath, layer) {
                    continue
                }

                // Check if this layer is allowed
                allowed := false
                for _, allowedLayer := range allowedLayers {
                    if layer == allowedLayer {
                        allowed = true
                        break
                    }
                }

                if !allowed {
                    return fmt.Errorf(
                        "%s: %s cannot import from %s (layer violation)",
                        path, currentLayer, layer,
                    )
                }
            }
        }

        return nil
    })
}
```

## Running Architecture Tests

```bash
# Locally
go run ./tools/arch-test

# In CI (fails build on violations)
- name: Validate Architecture
  run: go run ./tools/arch-test

# Via mise
mise run arch:check
```

## Integration with CI

Architecture tests should be:
- **Required** - Block merge if tests fail
- **Fast** - Run on every PR (~5-10 seconds)
- **Clear** - Provide actionable error messages
- **Comprehensive** - Cover all architectural rules

Example CI failure message:
```
✗ Architecture validation failed

Checking: service-isolation
  ✗ FAILED: services/servicebsvc/internal/adapters/outbound/helper.go
    imports services/serviceasvc/internal/domain/entitya
    (cross-service internal import)

Fix: Remove direct import of serviceasvc internals.
      Use bridge/serviceasvc instead.
```

## Additional Validation Rules

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
