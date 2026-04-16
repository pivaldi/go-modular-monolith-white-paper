# Failure Modes

Even with a well-designed architecture, common anti-patterns can undermine its benefits. Recognize and avoid these failure modes:

## Contract Definition Bloat

**Problem:** Contract packages accumulate business logic, validation, or utilities instead of staying as pure generated interfaces.

**Symptoms:**
- Contract DTOs contain validation logic
- Helper functions performing calculations in contract package
- Contract packages importing external dependencies (databases, HTTP clients)
- Multiple services depending on contract utilities for shared behavior

**Example:**
```go
// ✗ BAD: Business logic in contract package
package todo

func (dto *TaskDTO) IsOverdue() bool {
    return dto.DueDate.Before(time.Now())  // Business rule in contract!
}

func CalculateCompletionRate(total, done int) float64 {
    return float64(done) / float64(total)  // Shared utility in contract!
}
```

**Solution:**
- Keep contract packages as pure generated interfaces + event topics + error codes
- Move validation to the domain layer
- Duplicate simple utilities rather than sharing through contracts
- If logic is needed in multiple modules, it belongs in a shared library (`libs/ogl`), not a contract package

**Why it happens:** Teams see contracts as "shared" and use them for convenience, creating coupling.

## Over-Modularization

**Problem:** Creating too many small services or modules too early, before domain boundaries are understood.

**Symptoms:**
- Modules that always deploy together
- Frequent changes requiring updates to multiple modules
- Circular communication patterns (A calls B, B calls A)
- More contract packages than feature modules
- Single responsibilities that could be methods on an existing aggregate

**Example:**
```
✗ BAD: Too granular
modules/
├── todo-creation/       # Just creates todos
├── todo-validation/     # Just validates todos
├── todo-notification/   # Just sends notifications
└── todo-storage/        # Just stores todos

✓ GOOD: Cohesive boundary
modules/
└── todo/                # Manages todo lifecycle
    └── internal/
        ├── domain/
        ├── application/
        └── adapters/
```

**Solution:**
- Start with fewer, larger modules based on business capabilities
- Let module boundaries emerge from understanding the domain
- Use `internal/` packages for organization within a module
- Only create a new module when you understand the boundary and have a reason to deploy separately

**Why it happens:** Microservice enthusiasm, misunderstanding of DDD bounded contexts, premature optimization.

## Premature Service Extraction

**Problem:** Splitting modules into separate deployments before understanding operational requirements or communication patterns.

**Symptoms:**
- Network calls for operations that should be transactional
- Performance degradation after extraction
- Distributed transaction complexity
- Teams spending more time on infrastructure than features

**Example:**
```go
// ✗ PROBLEM: Extracted too early, now need distributed transaction
func CreateTodo(ctx context.Context, req CreateTodoRequest) error {
    // Network call
    user, err := authClient.ValidateUser(ctx, req.UserID)
    if err != nil {
        return err
    }

    // Network call
    todo, err := todoClient.CreateTodo(ctx, req)
    if err != nil {
        return err  // Auth check succeeded but todo failed — inconsistent!
    }

    // Network call
    err = notifClient.NotifyCreated(ctx, todo.ID)
    if err != nil {
        // What now? Rollback todo creation?
    }

    return nil
}
```

**Solution:**
- Keep modules in-process (via contract interfaces + in-process accessors) until you have a clear reason to separate
- Understand transactional boundaries before extracting
- Use architecture tests (`mmw check arch`) to enforce boundaries without deploying separately
- Extract only when you need independent scaling, deployment, or team ownership

**Why it happens:** Pressure to "do microservices," resume-driven development, misunderstanding that boundaries ≠ deployment.

## Contract-vs-Port Confusion

**Problem:** Teams confuse when to use external contract interfaces vs. internal application ports.

**Symptoms:**
- Application layer directly importing contract packages instead of using adapters
- Port interfaces duplicating contract interfaces exactly
- Tight coupling to contract DTOs in the domain layer

**Example:**
```go
// ✗ BAD: Application layer directly using contract package
package command

import (
    defauth "github.com/pivaldi/mmw-contracts/go/application/auth"  // Wrong layer!
)

type CreateTodoCommand struct {
    authSvc defauth.AuthPrivateService  // Directly coupled to contract package
}
```

**Solution:**
- **Contract package** = Module-to-module boundary (external API, generated from proto)
- **Port** = Application-to-adapter boundary (internal interface, hand-written)
- Application layer only knows about ports, never about contract packages
- Adapters implement ports and may use contract packages internally

**Correct flow:**
```
Application → Port (owns) → Adapter (implements) → Contract package (uses) → Other Module
```

**Why it happens:** Misunderstanding hexagonal architecture, treating contract packages as "just another dependency."

## Ignoring the Compiler

**Problem:** Bypassing module boundaries through reflection, build tags, or repository structure tricks.

**Symptoms:**
- Using reflection to access internal types
- Build tag hacks to share code
- Git submodules to work around go.mod restrictions
- Complaints that "Go modules are too restrictive"

**Example:**
```go
// ✗ BAD: Using reflection to bypass boundaries
func GetInternalState(service interface{}) map[string]interface{} {
    val := reflect.ValueOf(service).Elem()
    // Access unexported fields — defeats the entire purpose of internal/
}
```

**Solution:**
- If the compiler prevents it, that's the architecture working correctly
- Redesign the boundary rather than bypassing it
- Make internal APIs public through the contract package if truly needed
- Trust that boundaries prevent coupling for good reason

**Why it happens:** Impatience, viewing boundaries as obstacles rather than design constraints.

## Skipping Architecture Tests

**Problem:** Not implementing or enforcing architectural rules with automated tests.

**Symptoms:**
- Module boundaries erode over time
- "Just this once" cross-module imports through `internal/`
- Domain layer importing infrastructure "temporarily"
- Contract packages growing dependencies
- Discovered during code review instead of CI

**Solution:**
- Use `mmw check arch` (backed by `mmw/pkg/archtest/`) in CI — fail the build on violations
- Treat architectural violations like failing tests — must fix immediately
- Review and update architectural rules as the system evolves

The `mmw check arch` command runs all registered `Validator` implementations
and reports violations with file and import details. See [arch-test.md](arch-test.md).

**Why it happens:** "It compiles, ship it" mentality, treating architecture as documentation instead of enforceable design.

## Broken Module Interface

**Symptom:** Build error: `*Module does not implement core.Module`

**Cause:** A module's `Start` method has the wrong signature, is missing,
or is on a value receiver instead of a pointer receiver.

**Prevention:** The compile-time assertion in every module file:

```go
var _ pfcore.Module = (*Module)(nil)
```

This line makes the problem a build error rather than a runtime panic.
The build fails immediately at `cmd/mmw/main.go` when the modules slice
is constructed.

**Wrong (won't satisfy interface):**
```go
func (m Module) Start(ctx context.Context) error { ... }   // value receiver
func (m *Module) Run(ctx context.Context) error { ... }    // wrong method name
```

**Right:**
```go
func (m *Module) Start(ctx context.Context) error { ... }  // pointer receiver, correct name
```
