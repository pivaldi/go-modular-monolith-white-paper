# Handling Cross-Service Events

While **Contract Definitions** handle synchronous Request/Response calls (Queries & Commands), **Events** represent asynchronous facts that happened in the past (e.g., `ServiceACreated`, `PaymentFailed`).

To maintain strict decoupling, we follow a specific pattern: **The Contract Definition holds the News (DTO), but the Service holds the Radio (Adapter).**

## The 4-Step Pattern

### 1. Define the Event in the Contract Definition
The event structure is part of the public contract. It belongs in the Contract Definition so consumers can understand the data structure without importing the producer's internal code.

```go
// contracts/definitions/serviceasvc/dto.go
package serviceasvc

import "time"

// ServiceACreatedEvent is a public fact.
// It lives here so other services can subscribe to it safely.
type ServiceACreatedEvent struct {
    ServiceAID string    `json:"servicea_id"`
    Email      string    `json:"email"`
    OccurredAt time.Time `json:"occurred_at"`
}
```

### 2. Define the Port in the Producer (Application Layer)
The domain layer needs to publish the event but shouldn't know *how* (Memory, Redis, Kafka).

```go
// services/serviceasvc/internal/application/ports/events.go
type EventPublisher interface {
    PublishServiceACreated(ctx context.Context, event serviceasvc.ServiceACreatedEvent) error
}
```

### 3. Implement the Adapter in the Producer (Infrastructure)
This is where the logic lives. For the modular monolith, we typically use an in-memory bus (like Watermill's GoChannel) or a simple channel.

```go
// services/serviceasvc/internal/adapters/outbound/eventbus/memory.go
func (p *InMemoryPublisher) PublishServiceACreated(ctx context.Context, event serviceasvc.ServiceACreatedEvent) error {
    // Logic lives here (marshaling, sending to channel), NOT in the contract definition.
    payload, _ := json.Marshal(event)
    return p.bus.Publish("servicea-events", payload)
}
```

### 4. Wire it in `main.go`
The Composition Root acts as the infrastructure glue. It creates the bus and connects the services.

```go
// cmd/monolith/main.go
func main() {
    // 1. Infrastructure: Create the shared bus
    eventBus := watermill.NewGoChannel()

    // 2. Producer: Inject Publisher Adapter
    serviceaPub := servicea_eventbus.NewInMemoryPublisher(eventBus)
    serviceasvc := serviceasvc.New(..., serviceaPub)

    // 3. Consumer: Subscribe and handle
    // The consumer service (Service B) implements a handler adapter
    msgs, _ := eventBus.Subscribe(context.Background(), "servicea-events")
    go servicebsvc_adapter.HandleServiceAEvents(msgs, servicebService)

    // ...
}
```

## Why this fits the Architecture

1.  **Pure Contract Definition:** The contract definition only contains the `struct`. It has no logic, no dependencies on event libraries, and no `Publish()` methods.
2.  **Decoupling:** The Consumer (Service B) imports `contracts/definitions/serviceasvc` to know what the event looks like, but never imports `services/serviceasvc/internal`.
3.  **Migration Ready:** To switch to Microservices:
    *   Change `InMemoryPublisher` to `KafkaPublisher` in the Producer.
    *   Change `main.go` wiring.
    *   The Domain Logic and Contract DTOs remain untouched.

## Anti-Pattern Warning

** ✗ Don't put the Publisher interface in the Contract Definition.**

```go
// BAD: contracts/definitions/serviceasvc/api.go
type EventPublisher interface {
    Publish(...) // Don't do this!
}
```

**Why?**
*   The Contract Definition defines the **Service API** (Inbound).
*   The Publisher is a **Dependency** (Outbound).
*   Putting outbound interfaces in the contract definition couples consumers to the producer's infrastructure needs.
*   Keep the `EventPublisher` interface inside the **Producer's Application Layer** (`internal/application/ports`).
