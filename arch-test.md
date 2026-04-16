# Architecture Testing

## Implementation

The architecture validator lives in `mmw/pkg/archtest/` and is invoked via the `mmw check arch` CLI command.

```go
// mmw/pkg/archtest/archtest.go
package archtest

// RunAll runs all architectural validators from repoRoot and returns exit code (0=pass, 1=fail).
func RunAll(repoRoot string) int {
    rep := reporter.NewReporter(os.Stdout)
    rep.PrintHeader("Architecture Validation")

    // Per-module arch:check tasks via mise
    services, err := orchestrator.DiscoverServices(repoRoot+"/modules", "arch:check")
    for _, svc := range services {
        result := orchestrator.RunServiceCheck(svc.Path, svc.Name)
        rep.PrintCheck(svc.Name, "Validating service architecture boundaries", err)
    }

    // Cross-cutting validators
    validators := []Validator{
        &custom.ContractPurityValidator{
            ContractsDir: repoRoot + "/contracts/definitions",
            RepoRoot:     repoRoot,
        },
        &custom.LibDependencyValidator{
            LibsDir:  repoRoot + "/libs",
            MmwDir:   repoRoot + "/mmw",
            RepoRoot: repoRoot,
        },
        &custom.DomainPurityValidator{ModulesDir: repoRoot + "/modules"},
        &custom.ApplicationPurityValidator{ModulesDir: repoRoot + "/modules"},
    }
    for _, v := range validators {
        rep.PrintCheck(v.Name(), v.Description(), v.Check())
    }

    return rep.Summary()
}
```

Built-in validators:

| Validator | Rule |
|---|---|
| `ContractPurityValidator` | `contracts/go/application/` must not import application or infrastructure packages |
| `LibDependencyValidator` | `libs/ogl` and `mmw/` must not import module-specific code |
| `DomainPurityValidator` | `internal/domain/` must not import adapters, infra, or application packages |
| `ApplicationPurityValidator` | `internal/application/` must not import adapters or infra packages |

Per-module checks are discovered via `mise run arch:check` in each module directory. They use `arch-go` for intra-module layer validation.

## Compile-Time Interface Assertion

The first-line architecture check is a compile-time assertion in every module file:

```go
// modules/todo/todo.go
var _ pfcore.Module = (*Module)(nil)
```

If `Start(ctx context.Context) error` is missing or has the wrong signature,
the build fails immediately — before any runtime test runs.

**Wrong (won't satisfy interface):**
```go
func (m Module) Start(ctx context.Context) error { ... }   // value receiver
func (m *Module) Run(ctx context.Context) error { ... }    // wrong method name
```

**Right:**
```go
func (m *Module) Start(ctx context.Context) error { ... }  // pointer receiver, correct name
```

## Running Architecture Checks

```bash
# Via mmw-cli
mmw check arch

# Via mise (from workspace root)
mise run arch:check

# Direct
go run ./mmw/cmd/mmw-cli check arch
```

## Example Output

```
Architecture Validation
────────────────────────────────────────────
✓ todo         Validating service architecture boundaries
✓ auth         Validating service architecture boundaries
✓ ContractPurity     Contract definitions must not import application or infra packages
✓ LibDependency      Shared libs must not import module-specific code
✓ DomainPurity       Domain layer must not import adapters/infra/application
✓ ApplicationPurity  Application layer must not import adapters/infra
────────────────────────────────────────────
All checks passed.
```

## Integration with CI

```yaml
arch-check:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-go@v5
      with:
        go-version: '1.26'
    - run: go build -o /usr/local/bin/mmw ./mmw/cmd/mmw-cli
      working-directory: poc
    - run: mmw check arch
      working-directory: poc
```

Example CI failure message:
```
✗ Architecture validation failed

✗ todo    Validating service architecture boundaries
  modules/todo/internal/adapters/connect/handler.go
    imports modules/auth/internal/domain (cross-module internal import)

Fix: Remove direct import of auth internals.
     Depend on contracts/go/application/auth instead.
```
