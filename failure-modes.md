# Failure Modes

Even with a well-designed architecture, common anti-patterns can undermine its benefits. Recognize and avoid these failure modes:

## Bridge Bloat

**Problem:** Bridge modules accumulate business logic, validation, or utilities instead of staying pure interfaces.

**Symptoms:**
- Bridge DTOs contain validation logic
- Helper functions performing calculations in bridge package
- Bridge modules importing external dependencies (databases, HTTP clients)
- Multiple services depending on bridge utilities for shared behavior

**Example:**
```go
// ✗ BAD: Business logic in bridge
package authorsvc

func (dto *AuthorDTO) IsPopular() bool {
    return dto.FollowerCount > 10000  // Business rule in bridge!
}

func CalculateRating(articles, followers int) float64 {
    return float64(articles) * 0.3 + float64(followers) * 0.7  // Shared utility
}
```

**Solution:**
- Keep bridges as pure data transfer and interface definitions
- Move validation to domain layer
- Duplicate simple utilities rather than sharing through bridges
- If logic is needed in multiple services, it belongs in a shared library, not a bridge

**Why it happens:** Teams see bridges as "shared" and use them for convenience, creating coupling.

## Over-Modularization

**Problem:** Creating too many small services or modules too early, before domain boundaries are understood.

**Symptoms:**
- Services that always deploy together
- Frequent changes requiring updates to multiple services
- Circular communication patterns (A calls B, B calls A)
- Services with single responsibilities that could be functions
- More bridge modules than services

**Example:**
```
✗ BAD: Too granular
services/
├── user-creation-svc/      # Just creates users
├── user-validation-svc/    # Just validates users
├── user-notification-svc/  # Just sends user emails
└── user-storage-svc/       # Just stores users

✓ GOOD: Cohesive boundary
services/
└── usersvc/                # Manages user lifecycle
    └── internal/
        ├── domain/
        ├── application/
        └── adapters/
```

**Solution:**
- Start with fewer, larger services based on business capabilities
- Let service boundaries emerge from understanding the domain
- Use internal packages for organization within a service
- Only create a new service when you understand the boundary and have a reason to deploy separately

**Why it happens:** Microservice enthusiasm, misunderstanding of DDD bounded contexts, premature optimization.

## Premature Service Extraction

**Problem:** Splitting services into separate deployments before understanding operational requirements or communication patterns.

**Symptoms:**
- Network calls for operations that should be transactional
- Performance degradation after extraction
- Distributed transaction complexity
- Frequent "rollback" discussions to merge services back
- Teams spending more time on infrastructure than features

**Example:**
```go
// ✗ PROBLEM: Extracted too early, now need distributed transaction
func CreateArticle(ctx context.Context, req CreateArticleRequest) error {
    // Network call
    author, err := authorClient.GetAuthor(ctx, req.AuthorID)
    if err != nil {
        return err
    }

    // Network call
    article, err := articleClient.CreateArticle(ctx, req)
    if err != nil {
        return err  // Author check succeeded but article failed - inconsistent!
    }

    // Network call
    err = notificationClient.NotifyFollowers(ctx, article.ID)
    if err != nil {
        // What now? Rollback article creation?
    }

    return nil
}
```

**Solution:**
- Keep services in-process (using bridges) until you have a clear reason to separate
- Understand transactional boundaries before extracting
- Use architecture tests to enforce boundaries without deploying separately
- Extract only when you need independent scaling, deployment, or team ownership

**Why it happens:** Pressure to "do microservices," resume-driven development, misunderstanding that boundaries ≠ deployment.

## Bridge-Internal Confusion

**Problem:** Teams confuse when to use bridge interfaces vs. internal application ports.

**Symptoms:**
- Application layer directly importing bridge modules instead of using adapters
- Port interfaces duplicating bridge interfaces exactly
- Confusion about "which interface to use"
- Tight coupling to bridge DTOs in domain layer

**Example:**
```go
// ✗ BAD: Application layer directly using bridge
package command

import (
    "github.com/.../bridge/authorsvc"  // Wrong layer!
)

type CreateArticleCommand struct {
    authorService authorsvc.AuthorService  // Directly coupled to bridge
}
```

**Solution:**
- **Bridge** = Service-to-service boundary (external API)
- **Port** = Application-to-adapter boundary (internal API)
- Application layer only knows about ports, never about bridges
- Adapters implement ports and may use bridges internally

**Correct flow:**
```
Application -> Port (owns) -> Adapter (implements) -> Bridge (uses) -> Other Service
```

**Why it happens:** Misunderstanding hexagonal architecture, treating bridges as "just another dependency."

## Ignoring the Compiler

**Problem:** Bypassing module boundaries through reflection, build tags, or repository structure tricks.

**Symptoms:**
- Using reflection to access internal types
- Build tag hacks to share code
- Git submodules to work around go.mod restrictions
- "Helper" packages outside module structure
- Complaints that "Go modules are too restrictive"

**Example:**
```go
// ✗ BAD: Using reflection to bypass boundaries
func GetInternalState(service interface{}) map[string]interface{} {
    val := reflect.ValueOf(service).Elem()
    // Access unexported fields...
}
```

**Solution:**
- If the compiler prevents it, that's the architecture working correctly
- Redesign the boundary rather than bypassing it
- Make internal APIs public through bridge if truly needed
- Trust that boundaries prevent coupling for good reason

**Why it happens:** Impatience, viewing boundaries as obstacles rather than design constraints.

## Skipping Architecture Tests

**Problem:** Not implementing or enforcing architectural rules with automated tests.

**Symptoms:**
- Service boundaries erode over time
- "Just this once" cross-service imports
- Domain layer importing infrastructure "temporarily"
- Bridge modules growing dependencies
- Discovered during code review instead of CI

**Solution:**
- Implement `tools/arch-test` as shown in this white paper
- Run architecture tests in CI (fail the build on violations)
- Treat architectural violations like failing tests - must fix immediately
- Review and update architectural rules as the system evolves

**Why it happens:** "It compiles, ship it" mentality, treating architecture as documentation instead of enforceable design.

