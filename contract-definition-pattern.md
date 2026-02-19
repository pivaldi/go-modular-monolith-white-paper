# Contract Definition Pattern

## Table of Contents

<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-refresh-toc -->
- [Contract Definition Pattern](#contract-definition-pattern)
  - [Table of Contents](#table-of-contents)
  - [Detailed Contract Architecture](#detailed-contract-architecture)
    - [Contract Definition Responsibilities](#contract-definition-responsibilities)
    - [Example: Service A Contract](#example-service-a-contract)
    - [Understanding InprocServer and InprocClient](#understanding-inprocserver-and-inprocclient)
    - [Using the Contract in Another Service](#using-the-contract-in-another-service)
    - [Wiring in main.go](#wiring-in-maingo)
  - [Contract-Based Architecture Benefits](#contract-based-architecture-benefits)
  - [Contract Anti-Patterns](#contract-anti-patterns)
    - [What's Now Impossible (Enforced by arch-test)](#whats-now-impossible-enforced-by-arch-test)
    - [What's Still Dangerous (Human Element)](#whats-still-dangerous-human-element)
    - [What Belongs Where](#what-belongs-where)
    - [Deep Dive: Why This Complexity?](#deep-dive-why-this-complexity)
      - [1. The "Dependency Hell" Problem](#1-the-dependency-hell-problem)
      - [2. Contract Sharing vs. Code Sharing](#2-contract-sharing-vs-code-sharing)
      - [3. The "Pure Contract Definition" Rule](#3-the-pure-contract-definition-rule)
    - [Deep Dive: Why Not Co-locate the API?](#deep-dive-why-not-co-locate-the-api)
      - [1. The "Dependency Hell" Trap (Source-Level Isolation)](#1-the-dependency-hell-trap-source-level-isolation)
      - [2. Prevention of Cyclic Dependencies](#2-prevention-of-cyclic-dependencies)
      - [3. Physical vs. Logical Boundaries](#3-physical-vs-logical-boundaries)
      - [Summary Comparison](#summary-comparison)
  - [Annexe: Why `/contracts/definitions/xxx` instead of `/service/api/xxx`?](#annexe-why-contractsdefinitionsxxx-instead-of-serviceapixxx)
<!-- markdown-toc end -->

## Detailed Contract Architecture

**The contract definition is the key innovation that enables strong boundaries with flexible transport.**

### Contract Definition Responsibilities

A contract definition for a service contains ONLY public contracts:
1. **Defines the public API** with Go interfaces
2. **Defines DTOs** ([data transfer objects](https://martinfowler.com/eaaCatalog/dataTransferObject.html))
3. **Defines public error types**
4. **Provides in-process client** that calls the interface (thin wrapper)

The service itself (not the contract definition) provides:
5. **In-process server implementation** in `services/*/internal/adapters/inbound/contracts/` that implements the contract interface

### Example: Service A Contract

See [example-servicea-contracts.md](example-servicea-contracts.md)

### Understanding InprocServer and InprocClient

Before diving into the full implementation, let's understand what these components are and how they work together.

**What They Are:**

- **InprocServer**: An inbound adapter that lives in `services/serviceasvc/internal/adapters/inbound/contracts/` (adapter pattern directory) and implements the contract interface by wrapping the service's application layer
- **InprocClient**: A thin client wrapper that lives in `contracts/definitions/serviceasvc/` and calls any implementation of the contract interface (no network)

**Simplified Structure:**

```go
// Contract Definition: contracts/definitions/serviceasvc/inproc_client.go
// InprocClient accepts ANY implementation of ServiceA
type InprocClient struct {
    server ServiceA  // Interface reference (not concrete type!)
}

// Service Adapter: services/serviceasvc/internal/adapters/inbound/contracts/inproc_server.go
// InprocServer implements the contract interface
type InprocServer struct {
    // References to the service's internal application layer
    getEntityAQuery    *query.GetEntityAQuery      // from serviceasvc/internal/application/query
    listEntityAQuery  *query.ListEntityAQuery    // from serviceasvc/internal/application/query
    createEntityACmd   *command.CreateEntityACommand  // from serviceasvc/internal/application/command
    updateEntityACmd   *command.UpdateEntityACommand  // from serviceasvc/internal/application/command
}

// Factory returns interface type for loose coupling
func NewInprocServer(...) ServiceA {
    return &InprocServer{...}
}
```

**The Flow:**

```mermaid
graph TB
    subgraph consumer["SERVICEBSVC (Consumer Service)"]
        app_layer["Application Layer"]
        port["ServiceAClient Port<br/>(interface)"]
        inproc_adapter["InprocAdapter"]

        app_layer -->|needs service A data| port
        port --> inproc_adapter
    end

    subgraph contract["CONTRACT (contracts/definitions/serviceasvc)"]
        inproc_client["InprocClient<br/>(thin wrapper)"]
        interface["ServiceA<br/>(interface)"]

        inproc_client -->|implements| interface
    end

    subgraph provider["SERVICEASVC (Provider Service)"]
        inproc_server["InprocServer<br/>(internal/adapters/inbound/contracts)"]
        query["GetEntityAQuery.Execute"]
        domain["Domain Layer"]
        repo["Repository"]

        inproc_server -->|implements ServiceA<br/>CAN import internal| query
        query --> domain
        domain --> repo
    end

    inproc_adapter -->|calls| inproc_client
    inproc_client -->|calls via interface| inproc_server

    style consumer fill:#e1f5ff
    style contract fill:#fff4e1
    style provider fill:#ffe1e1
```

**Lifecycle:**

1. **Monolith main.go** (direct explicit wiring):
   ```go
   // cmd/monolith/main.go
   package main

   import (
       contractservicea "github.com/example/service-manager/contracts/definitions/serviceasvc"
       serviceaconfig "github.com/example/service-manager/services/serviceasvc/config"
       serviceaadapters "github.com/example/service-manager/services/serviceasvc/internal/adapters/inbound/contracts"
       "github.com/example/service-manager/services/servicebsvc"
   )

   func main() {
       ctx := context.Background()

       // Phase 1: Load service A config
       serviceaCfg, _ := serviceaconfig.Load()

       // Phase 2: Create InprocServer (lives in serviceasvc internal adapters)
       // Returns: contractservicea.ServiceA interface
       serviceAServer := serviceaadapters.NewInprocServer(serviceaCfg, logger)

       // Phase 3: Wrap with InprocClient (lives in contract definition)
       // Accepts: contractservicea.ServiceA interface
       serviceAClient := contractservicea.NewInprocClient(serviceAServer)

       // Phase 4: Initialize consumer service with the client
       servicebCfg, _ := servicebconfig.Load()
       serviceBService := servicebsvc.New(servicebCfg, serviceAClient)

       // Phase 5: Run services via errgroup supervisor
       g, gCtx := errgroup.WithContext(ctx)
       g.Go(func() error { return serviceAServer.Run(gCtx) })
       g.Go(func() error { return serviceBService.Run(gCtx) })

       if err := g.Wait(); err != nil {
           log.Fatal(err)
       }
   }
   ```

2. **Runtime Call Flow**:
   ```go
   // servicebsvc application layer
   result := serviceAClient.GetEntityA(ctx, "entity-123")
       │
       ▼ (interface call)
   // servicebsvc outbound adapter
   inprocAdapter.GetEntityA(ctx, "entity-123")
       │
       ▼ (interface call)
   // contract InprocClient (thin wrapper in contract definition)
   client.server.GetEntityA(ctx, "entity-123")
       │
       ▼ (interface call - no knowledge of concrete InprocServer type!)
   // serviceasvc InprocServer (inbound adapter in service internal)
   server.getEntityAQuery.Execute(ctx, "entity-123")
       │
       ▼ (direct function call - same module, can import internals)
   // serviceasvc internal application layer
   query.Execute(ctx, "entity-123")
   ```

**Key Principles:**

1. **True Module Independence**:
   - InprocServer lives in `services/serviceasvc/internal/adapters/inbound/contracts/`
   - Contract definition has literally zero dependencies (no `require` statements)
   - InprocServer CAN import service internals (same Go module)
   - Contract definition CANNOT import service internals (different Go module - compiler enforced)

2. **Interface-Based Coupling**:
   - InprocClient holds `ServiceA` interface, not `*InprocServer` concrete type
   - NewInprocServer() returns `ServiceA` interface
   - Enables loose coupling and testability
   - Consumers never know about the concrete implementation

3. **Thin Delegation Layer**:
   - InprocServer contains NO business logic
   - It only translates between contract DTOs and internal types
   - It maps domain errors to contract errors
   - Pure adapter pattern

4. **Zero Network Overhead**:
   - InprocClient -> InprocServer is a direct function call via interface
   - No serialization, no HTTP, no network latency
   - Performance identical to direct internal imports (but with boundaries!)

5. **Swappable Implementation**:
   - Consumer sees only the contract interface (`ServiceA`)
   - Can swap InprocClient for NetworkClient without changing application layer
   - Contract definition provides the abstraction point

**Complete Implementation:**

Now see the full implementation with all methods and error handling in the file [example-contracts-serviceasvc.md](example-contracts-serviceasvc.md).

### Using the Contract in Another Service

```go
//services/servicebsvc/internal/adapters/outbound/serviceaclient/inproc/client.go
package inproc

import (
    "context"

    // Import the contract definition (public, allowed)
    "github.com/example/service-manager/contracts/definitions/serviceasvc"

    // Import application port (internal to servicebsvc)
    "github.com/example/service-manager/services/servicebsvc/internal/application/ports"
)

// Client adapts the serviceasvc.ServiceA to our application's ports.ServiceAClient.
type Client struct {
    contract serviceasvc.ServiceA
}

func NewClient(contract serviceasvc.ServiceA) *Client {
    return &Client{contract: contract}
}

func (c *Client) GetEntityA(ctx context.Context, entityID string) (*ports.EntityA, error) {
    // Call contract (which calls serviceasvc internally)
    dto, err := c.contract.GetEntityA(ctx, entityID)
    if err != nil {
        // Translate contract errors to application errors
        if errors.Is(err, serviceasvc.ErrEntityANotFound) {
            return nil, ports.ErrEntityANotFound
        }
        return nil, ports.ErrServiceADown
    }

    // Map contract DTO to application DTO
    return &ports.EntityA{
        ID:        dto.ID,
        Name:      dto.Name,
        Bio:       dto.Bio,
        AvatarURL: dto.AvatarURL,
    }, nil
}
```

### Wiring in main.go

The monolith's main.go performs direct explicit wiring with clear initialization order:

```go
// cmd/monolith/main.go
package main

import (
    contractservicea "github.com/example/service-manager/contracts/definitions/serviceasvc"
    serviceaconfig "github.com/example/service-manager/services/serviceasvc/config"
    serviceaadapters "github.com/example/service-manager/services/serviceasvc/internal/adapters/inbound/contracts"
    servicebconfig "github.com/example/service-manager/services/servicebsvc/config"
    "github.com/example/service-manager/services/servicebsvc"
    serviceaclientadapter "github.com/example/service-manager/services/servicebsvc/internal/adapters/outbound/serviceaclient/inproc"
)

func main() {
    ctx := context.Background()

    // 1. Config: Load service A configuration
    serviceaCfg, _ := serviceaconfig.Load()

    // 2. Provider: Create InprocServer (returns interface)
    serviceAServer := serviceaadapters.NewInprocServer(serviceaCfg, logger)

    // 3. Contract: Wrap with InprocClient
    serviceAClient := contractservicea.NewInprocClient(serviceAServer)

    // 4. Consumer Adapter: Wrap contract client in outbound adapter
    serviceAClientAdapter := serviceaclientadapter.NewClient(serviceAClient)

    // 5. Consumer Service: Initialize with dependencies
    servicebCfg, _ := servicebconfig.Load()
    serviceBService := servicebsvc.New(servicebCfg, serviceAClientAdapter)

    // 6. Run: Start all services via errgroup
    g, gCtx := errgroup.WithContext(ctx)
    g.Go(func() error { return serviceAServer.Run(gCtx) })
    g.Go(func() error { return serviceBService.Run(gCtx) })

    if err := g.Wait(); err != nil {
        log.Fatal(err)
    }
}
```

**Key Points:**
- **Initialization order is explicit**: Provider service before consumer service
- **No registry needed**: Direct wiring in main.go
- **Type safety**: Compiler enforces correct interfaces
- **Interface-based**: Everything operates on `ServiceA` interface, not concrete types

## Contract-Based Architecture Benefits

1. **Compiler-Enforced Boundaries**
   - Service B cannot import `serviceasvc/internal` (compiler error)
   - Service B can only import `contracts/definitions/serviceasvc` (public API)
   - Violations are caught at compile time, not runtime or review

2. **Zero Network Overhead**
   - In-process client -> server is a direct function call
   - No serialization, no HTTP stack, no network latency
   - Performance equivalent to shared-module monolith

3. **Clear Migration Path**
   - Today: Service B uses `contracts/definitions/serviceasvc.InprocClient`
   - Tomorrow: Service B uses `serviceaconnect.Client` (HTTP/Connect)
   - Change is localized to wiring in `main.go`
   - Application layer is unchanged

4. **Explicit Seam**
   - Contract definition makes the service boundary visible
   - Clear "public API" vs "internal implementation"
   - Documentation target for service contracts

5. **Flexible Implementation**
   - Same interface works for multiple transports
   - Can mix transports (some in-process, some network)
   - Easy to test (mock the contract interface)

## Contract Anti-Patterns

**Note:** Our `arch-test` suite now physically prevents importing internal packages or adding external dependencies in contract definitions. However, **logical coupling** is still possible and must be caught in code review. Even if code compiles, it doesn't mean it belongs in a contract definition.

**Contract definitions must remain:**

- **Stateless** - No global variables, no caches, no state
- **Business-logic free** - No domain rules, no validation, not even "pure" helper functions
- **DTO + interface only** - Just data contracts and method signatures

### What's Now Impossible (Enforced by arch-test)

These violations will be caught by `arch-test` in CI and fail the build:

✗ **Contract definition importing service internals**
```go
// BAD: Compiler allows, but arch-test prevents
package serviceasvc

import "github.com/.../services/serviceasvc/internal/domain/entitya"

type ServiceA interface {
    GetEntityA(ctx context.Context, id string) (*entitya.EntityA, error)
    // arch-test detects internal/ import and FAILS
}
```

✗ **Contract definition with external dependencies**
```go
// BAD: Adding dependencies in contracts/definitions/serviceasvc/go.mod
module github.com/example/service-manager/contracts/definitions/serviceasvc

require (
    github.com/google/uuid v1.3.0  // arch-test detects require and FAILS
)
```

✓ **What arch-test enforces:**
- Contract definition `go.mod` has **zero** `require` statements
- Contract definition code never imports any `internal/` packages
- Violations fail CI before merge

### What's Still Dangerous (Human Element)

These violations compile successfully but break architectural principles. They must be caught in code review:

✗ **Contract bloat with business logic**
```go
// BAD: Compiles fine, but violates contract definition purity
package serviceasvc

func (dto *EntityA) Validate() error {
    if len(dto.Name) < 3 {
        return errors.New("name too short")  // Domain logic doesn't belong here
    }
    // Even though this is "pure" logic with no dependencies,
    // it creates coupling - belongs in serviceasvc/internal/domain
}

func CalculateServiceARating(articles int, followers int) int {
    return articles*10 + followers  // Pure calculation, but still wrong place!
}
```

**Why this is dangerous:**
- Creates logical coupling across services that import the contract definition
- Violates Single Responsibility - contract definitions are contracts, not business logic
- Even "pure" helper functions create shared-kernel problems

✓ **Keep contract definitions pure:**
```go
// GOOD: Contract definition is just a contract
type EntityA struct {
    ID   string
    Name string
    Bio  string
}

// Validation and business logic live in serviceasvc/internal/domain
// Contract definitions only define the contract
```

✗ **Shared utilities in contract definition**
```go
// BAD: Compiles, but creates service coupling
package serviceasvc

import "time"

func FormatEntityADate(t time.Time) string {
    return t.Format("2006-01-02")  // Seemingly innocent utility
    // Problem: Multiple services now depend on this formatting logic
}
```

**Why this is wrong:**
- If formatting changes, ALL services importing this contract definition are affected
- You've recreated a shared-kernel monolith with loose coupling
- Each service should handle its own formatting needs

✗ **Contract-Internal confusion (DTOs as Entities)**
```go
// BAD: Domain layer using contract DTOs directly
package domain

import "github.com/.../contracts/definitions/serviceasvc"

type User struct {
    ID      string
    EntityA  *serviceasvc.EntityA  // Domain using DTO as entity - WRONG
}
```

**Why this is dangerous:**
- Domain layer becomes coupled to external contract changes
- DTOs are for transport, Entities are for business logic
- Breaks dependency inversion (domain depends on adapter contract)

✓ **Proper layering:**
```go
// GOOD: Domain has its own types
package domain

type EntityA struct {  // Domain entity
    ID   EntityAID
    Name EntityAName
}

// Adapter layer maps between domain entities and contract DTOs
// services/servicebsvc/internal/adapters/outbound/serviceaclient/inproc/client.go
func (c *Client) GetEntityA(id string) (*domain.EntityA, error) {
    dto, err := c.contract.GetEntityA(ctx, id)
    // Map DTO → Domain Entity
    return &domain.EntityA{
        ID:   domain.NewEntityAID(dto.ID),
        Name: domain.NewEntityAName(dto.Name),
    }, nil
}
```

**The Golden Rule:**

> If you're tempted to add ANY logic to a contract definition (even "pure" helper functions), you're recreating a shared-kernel monolith. Stop and refactor the logic into the appropriate service's domain layer instead.

### What Belongs Where

**In Contract Definitions** (`contracts/definitions/serviceasvc/`):
- Interface definitions (service contracts)
- DTOs (pure data structures, no methods except basic getters/setters)
- Error constants (semantic errors like `ErrNotFound`)
- InprocClient (thin wrapper that calls the interface)

**In Service Internal Adapters** (`services/*/internal/adapters/inbound/contracts/`):
- InprocServer (implements the contract interface)
- DTO mapping logic (contract DTOs ↔ domain entities)
- Error translation (domain errors → contract errors)

**In Service Domain Layer** (`services/*/internal/domain/`):
- Business validation rules
- Domain calculations and algorithms
- Entity behavior and invariants
- Value objects with rich behavior

**NOT in Contract Definitions or Adapters:**
- Business validation rules (→ domain layer)
- Domain calculations or algorithms (→ domain layer)
- Shared utilities across services (→ create separate shared library module if truly needed)
- Database models or repository logic (→ persistence adapters)
- HTTP handlers (→ HTTP inbound adapters)
- Configuration (→ service config package)
- Feature flags (→ service infrastructure)

By keeping contract definitions truly dependency-free and logically pure, you achieve true module independence and avoid the coupling problems that plague shared-kernel architectures.

### Deep Dive: Why This Complexity?

A common critique of this pattern is: *"Why not just use a single `go.mod` with strict linting? It's simpler."*

While a single module is simpler initially, the **Multiple Module (Workspace)** approach solves specific problems that linting cannot address.

#### 1. The "Dependency Hell" Problem
In Go, a single `go.mod` file resolves a **single dependency graph** for the entire project. This means every service in the monolith must share the exact same version of every library.

**Scenario:**
*   `Service A` relies on `aws-sdk-go` v1 (legacy).
*   `Service B` wants to use an AI library that requires `aws-sdk-go` v2.

**In a Single Module Monolith:**
You are blocked. You cannot upgrade Service B without refactoring Service A. One service's technical debt holds back the entire platform.

**In this Architecture:**
*   `services/serviceA/go.mod` requires `aws-sdk-go v1`
*   `services/serviceB/go.mod` requires `aws-sdk-go v2`

Because they are separate modules, they have **Independent Dependency Graphs**. The Go compiler handles this perfectly. This is critical for teams of 5-20 developers where coordinating library upgrades across the entire system is costly.

#### 2. Contract Sharing vs. Code Sharing
Another critique is that contract definitions create "Coupling." It is vital to distinguish between two types of coupling:

*   **Implementation Coupling (Bad):** Sharing logic, validators, or database helpers. This makes services fragile; changing one breaks the other.
*   **Contract Coupling (Necessary):** Sharing interfaces and DTOs.

Services **must** share a contract to communicate (whether it's JSON schemas, gRPC Protobufs, or Go Interfaces).

The contract definition is strictly **Contract Coupling**. It is the semantic equivalent of a shared `.proto` repository in gRPC, but for in-process Go communication. It contains **no logic**, only definitions.

#### 3. The "Pure Contract Definition" Rule
To ensure the contract definition remains a Contract and not a Shared Kernel, we enforce strict rules (via `arch-test`):

1.  **No Internal Imports:** The contract definition cannot import `services/xxx/internal`.
2.  **No Logic:** The implementation (`InprocServer`) lives inside the **Service**, not the contract definition.
3.  **Dependency Inversion:** The Service imports the contract definition to implement the interface. The contract definition depends on nothing.

This structure allows `Service A` to be extracted to a new repository by simply copying `services/serviceA` and `contracts/definitions/serviceA`, with zero entanglement with `Service B`.

### Deep Dive: Why Not Co-locate the API?

A frequent question from developers is: *"Why not just place the API package inside the service module (e.g., `services/serviceasvc/api`)? Why do we need a separate `contracts/definitions/` directory?"*

While co-locating the API appears simpler initially, it introduces severe architectural coupling in a Go environment.

#### 1. The "Dependency Hell" Trap (Source-Level Isolation)
If you place the API package inside `serviceasvc/api`, it shares the `go.mod` of the service.

**The Consequence:**
Any consumer (e.g., `Service B`) that imports `serviceasvc/api` must add a `require` for the **entire** `serviceasvc` module.

*   **Inherited Dependencies:** `Service B` implicitly inherits **ALL** of `Service A`'s dependencies (AWS SDKs, Database drivers, logging libs), even if it only needs a struct definition.
*   **Development Friction:** Running `go test ./...` in `Service B` forces the download and compilation of `Service A`'s heavy dependency tree.
*   **Version Conflicts:** If `Service A` relies on a legacy version of a library and `Service B` needs a modern version, you are blocked.

**With a Contract Definition:**
*   `contracts/definitions/serviceasvc` has a separate `go.mod` with **ZERO** dependencies.
*   `Service B` imports `contracts/definitions/serviceasvc` and inherits nothing.

*Note: While the final binary (`main.go`) will eventually merge all dependencies, the contract definition ensures **Source-Level Isolation**. This protects the development lifecycle, accelerates CI, and enables independent major version upgrades.*

#### 2. Prevention of Cyclic Dependencies
In complex systems, services often need to reference each other bi-directionally (e.g., *Orders* needs *Users* for addresses; *Users* needs *Orders* for history).

**If using `services/xxx/api`:**
*   Service A imports Service B's API.
*   Service B imports Service A's API.
*   **Result:** Go Module Cycle Error. Go does not allow cyclic module dependencies. You are structurally blocked.

**With Contract Definitions:**
*   Service A imports `contracts/definitions/ServiceB`.
*   Service B imports `contracts/definitions/ServiceA`.
*   **Result:** No cycle. The contract definitions are independent leaves in the dependency tree.

#### 3. Physical vs. Logical Boundaries
Moving the API into the service folder erodes the physical boundary that tooling can enforce.

*   **Current Architecture:** `arch-test` enforces that `contracts/definitions/` cannot import `internal/`. This is robust because they are different root directories.
*   **Co-located Approach:** If the API is in `services/serviceasvc/api`, developers are frequently tempted to move "helper" structs from `internal/` to `api/` for convenience, re-introducing coupling.

#### Summary Comparison

| Feature | Co-located API (`services/api`) | Contract Definition (`contracts/definitions/`) |
| :--- | :--- | :--- |
| **Dependency Graph** | **Coupled:** Consumer inherits Provider's entire `go.mod`. | **Decoupled:** Consumer inherits 0 dependencies. |
| **Bi-directional calls** | **Impossible:** Causes module cycles. | **Possible:** Contract definitions break the cycle. |
| **Migration** | **Hard:** Extracting the service means extracting the API consumers depend on. | **Easy:** The API (contract definition) is already separate. |
| **Clarity** | **Low:** Mixes Public Contract with Private Implementation. | **High:** Explicit separation of "What I do" vs "How I do it". |

**Conclusion:** Keep the API in the separate `contracts/definitions/` directory. It is the only way to guarantee the **Independent Dependency Graphs** that make this architecture scalable.

## Annexe: Why `/contracts/definitions/xxx` instead of `/service/api/xxx`?

This has been a long debate within our technical team…

At the question
> I would like to remove /contracts/definitions and move the content in the related service/api.

The answer is clear, **DON'T DO IT**. Here's why:

- **Compile-Time Dependencies VS Runtime Dependencies**
  - Importing `serviceA/api` into `serviceB` only gives `serviceB` access to the **Type Definitions (interfaces and DTOs)**. **It does not give `serviceB` a running instance of `serviceA`**.
  - If `serviceB`'s try to import `serviceA/api` and expect it to "just work," they will fail because the implementation behind the interface is missing, it's nil or non-existent.
  - If `serviceB` instantiates the real `serviceA`, `serviceB` would need to know how to configure `serviceA` (DB passwords, AWS keys, etc.).
    This breaks encapsulation and **Violates the Boundaries**.
  - The contract-based architecture solves this by enforcing **Dependency Inversion**.
- **Fatal flaw: Circular dependencies**
  - If Orders needs Users and Users needs Orders, `serviceA/api` ↔ `serviceA/api` creates a Go module cycle
  - Your build literally breaks - Go doesn't allow this
  - Contract definitions break the cycle because they're independent leaves
- **Real cost: Source-level pollution and "Dependency Hell" Trap**
  - Importing `serviceA/api` forces importing `serviceA`'s entire `go.mod` (pgx, AWS SDK, kafka-go, etc.)
  - go test in one service downloads dependencies from all services it talks to
  - CI becomes slower, local iteration becomes slower
  - Contract-based architecture solves this problem enforcing a dedicated layer `tests/e2e` for the tests and `cmd/monolith` as a bootstrap services' orchestrator, NOT inside `serviceB`'s codebase
- **When `services/api` works:**
  - Single-direction dependencies only (no cycles)
  - Small number of services (2-3)
  - Willing to accept slower tests/builds
  - The contract-based architecture solves this by exposing only interfaces and DTOs without any services' dependencies.
- **Physical Boundary Enforcement:**
  - Co-locating the API erodes the physical boundary.
  - It makes it significantly harder for tooling (like arch-test) to enforce that the public contract never imports internal implementation details.
- **Harder extraction later**: if `serviceB/api` lives inside the `serviceB` module, when distribuing a module, every consumer of this module must change imports and you often end up re-creating a shared "contracts" repo anyway.
- **Dependency leakage**: it becomes tempting for `serviceB/api` to pull in "just one helper" (logging, errors, DB types). Then consumers inherit `serviceB`'s dependency graph and coupling creeps back.
- **Weaker boundary semantics**: "API = public surface" + "service internals" live together; governance is harder than a dedicated contracts module.

Importing the API is necessary for the **code to compile**, but **it is insufficient for the code to run**.
Expecting `serviceB` to bootstrap `serviceA` for is an architectural anti-pattern that leads to **tight coupling** and the "Dependency Hell" that the contract-based architecture solves elegantly.
