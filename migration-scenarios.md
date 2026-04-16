## Migration Scenarios

### Scenario 1: From Traditional Monolith

**Starting point:** Single Go module, layered architecture

**Migration path:**

1. **Identify module boundaries** (1-2 weeks)
   - Analyze domain and find bounded contexts
   - Map existing code to future modules

2. **Extract first module** (2-4 weeks)
   - Create the module directory + `go.mod` + `go.work` entry
   - Move domain logic to the new module
   - Generate contracts via `protoc-gen-go-contracts`
   - Create adapters for dependencies
   - Test thoroughly

3. **Repeat for remaining modules** (incremental)
   - Extract one module at a time
   - Maintain a working system throughout

**Effort:** Medium (requires refactoring, but incremental)

### Scenario 2: From Single Module with Conventions

**Starting point:** One `go.mod`, convention-based boundaries

**Migration path:**

1. **Create proto definitions + generate contracts** (1 week)
   - Write `.proto` service definitions for each bounded context
   - Run `buf generate` with the standard plugin + `protoc-gen-go-contracts`
   - Contracts land in `contracts/go/application/{domain}/`

2. **Split into separate Go modules** (1-2 weeks)
   - Create `go.mod` per module
   - Add `go.work` to coordinate
   - Update imports to use contract packages

3. **Verify boundaries** (1 week)
   - Add architecture tests via `mmw check arch`
   - Fix any violations
   - Add to CI

**Effort:** Low-Medium (structure already exists, just formalize)

### Scenario 3: From Microservices (Consolidation)

**Starting point:** Separate repos, separate deployments

**Migration path:**

1. **Create monorepo** (1 week)
   - Move services to a single repo under `modules/`
   - Keep separate `go.mod` files
   - Add `go.work`

2. **Generate contract packages** (1-2 weeks)
   - Write `.proto` service definitions
   - Run `buf generate` + `protoc-gen-go-contracts`
   - Implement `connect_client.go` wrappers for network transport

3. **Switch to in-process** (1 week per module)
   - Update composition root (`cmd/mmw/main.go`) to wire modules directly
   - Use `authModule.PrivateService()` instead of an HTTP client
   - Test with both transports before removing network plumbing

**Effort:** Low (services already separated, just consolidate)

### Scenario 4: To Microservices (Distribution)

**Starting point:** This pattern (Go Workspaces + Contract Packages)

**Migration path:**

1. **Verify Connect handlers are wired** (1 day)
   - Each module already exposes a Connect handler via its HTTP server
   - Contracts already have `connect_client.go` wrappers (`defauth.NewPrivateHTTPClient`)

2. **Test network transport** (1 week)
   - In `cmd/mmw/main.go`, replace the in-process accessor with an HTTP client:
     ```go
     // Before (monolith):
     AuthSvc: authModule.PrivateService()

     // After (distributed):
     AuthSvc: defauth.NewPrivateHTTPClient(
         authv1connect.NewAuthPrivateServiceClient(&http.Client{}, "https://auth.internal"),
     )
     ```
   - No application code changes — only the composition root changes

3. **Deploy separately** (1 week per module)
   - Create deployment manifests
   - Deploy auth module to its own pod
   - Update the URL in the composition root
   - Monitor and iterate

**Effort:** Very Low (modules already isolated, Connect clients already generated)
