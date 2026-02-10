## Migration Scenarios

### Scenario 1: From Traditional Monolith

**Starting point:** Single Go module, layered architecture

**Migration path:**

1. **Identify service boundaries** (1-2 weeks)
   - Analyze domain and find bounded contexts
   - Map existing code to future services

2. **Extract first service** (2-4 weeks)
   - Create service module + bridge
   - Move domain logic to new service
   - Create adapters for dependencies
   - Test thoroughly

3. **Repeat for remaining services** (incremental)
   - Extract one service at a time
   - Maintain working system throughout

**Effort:** Medium (requires refactoring, but incremental)

### Scenario 2: From Single Module with Conventions

**Starting point:** One `go.mod`, convention-based boundaries

**Migration path:**

1. **Create bridge modules** (1 week)
   - Extract public APIs to bridge modules
   - Define interfaces and DTOs

2. **Split into separate Go modules** (1-2 weeks)
   - Create `go.mod` per service
   - Add `go.work` to coordinate
   - Update imports to use bridges

3. **Verify boundaries** (1 week)
   - Add arch-test tool
   - Fix any violations
   - Add to CI

**Effort:** Low-Medium (structure already exists, just formalize)

### Scenario 3: From Microservices (Consolidation)

**Starting point:** Separate repos, separate deployments

**Migration path:**

1. **Create monorepo** (1 week)
   - Move services to single repo
   - Keep separate modules
   - Add `go.work`

2. **Create bridge modules** (1-2 weeks)
   - Extract in-process interfaces
   - Implement InprocServer/Client
   - Keep Connect handlers

3. **Switch to in-process** (1 week per service)
   - Update wiring to use bridges
   - Test with both transports
   - Remove network calls for co-located services

**Effort:** Low (services already separated, just consolidate)

### Scenario 4: To Microservices (Distribution)

**Starting point:** This pattern (Go Workspaces + Bridges)

**Migration path:**

1. **Enable Connect handlers** (1 week)
   - Add protobuf contracts (if not using)
   - Implement Connect handlers (inbound)
   - Implement Connect clients (outbound)

2. **Test network transport** (1 week)
   - Switch service A to use Connect client
   - Keep services B, C on bridges
   - Verify functionality

3. **Deploy separately** (1 week per service)
   - Create deployment manifests
   - Deploy to separate hosts/pods
   - Update service URLs
   - Monitor and iterate

**Effort:** Very Low (services already isolated, adapters ready)
