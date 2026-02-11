# Handling Cross-Service Events

While **Bridge Modules** handle synchronous Request/Response calls (Queries & Commands), **Events** represent asynchronous facts that happened in the past (e.g., `AuthorCreated`, `PaymentFailed`).

To maintain strict decoupling, we follow a specific pattern: **The Bridge holds the News (DTO), but the Service holds the Radio (Adapter).**

## The 4-Step Pattern

### 1. Define the Event in the Bridge
The event structure is part of the public contract. It belongs in the Bridge so consumers can understand the data structure without importing the producer's internal code.

```go
// bridge/authorsvc/dto.go
package bridgeauthor

import "time"

// AuthorCreatedEvent is a public fact.
// It lives here so other services can subscribe to it safely.
type AuthorCreatedEvent struct {
    AuthorID   string    `json:"author_id"`
    Email      string    `json:"email"`
    OccurredAt time.Time `json:"occurred_at"`
}
```

### 2. Define the Port in the Producer (Application Layer)
The domain layer needs to publish the event but shouldn't know *how* (Memory, Redis, Kafka).

```go
// services/authorsvc/internal/application/ports/events.go
type EventPublisher interface {
    PublishAuthorCreated(ctx context.Context, event bridge.AuthorCreatedEvent) error
}
```

### 3. Implement the Adapter in the Producer (Infrastructure)
This is where the logic lives. For the modular monolith, we typically use an in-memory bus (like Watermill's GoChannel) or a simple channel.

```go
// services/authorsvc/internal/adapters/outbound/eventbus/memory.go
func (p *InMemoryPublisher) PublishAuthorCreated(ctx context.Context, event bridge.AuthorCreatedEvent) error {
    // Logic lives here (marshaling, sending to channel), NOT in the bridge.
    payload, _ := json.Marshal(event)
    return p.bus.Publish("author-events", payload)
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
    authorPub := author_eventbus.NewInMemoryPublisher(eventBus)
    authorSvc := authorsvc.New(..., authorPub)

    // 3. Consumer: Subscribe and handle
    // The consumer service (AuthSvc) implements a handler adapter
    msgs, _ := eventBus.Subscribe(context.Background(), "author-events")
    go authsvc_adapter.HandleAuthorEvents(msgs, authService)

    // ...
}
```

## Why this fits the Architecture

1.  **Pure Bridge:** The bridge module only contains the `struct`. It has no logic, no dependencies on event libraries, and no `Publish()` methods.
2.  **Decoupling:** The Consumer (AuthSvc) imports `bridge/authorsvc` to know what the event looks like, but never imports `services/authorsvc/internal`.
3.  **Migration Ready:** To switch to Microservices:
    *   Change `InMemoryPublisher` to `KafkaPublisher` in the Producer.
    *   Change `main.go` wiring.
    *   The Domain Logic and Bridge DTOs remain untouched.

## Anti-Pattern Warning

**❌ Don't put the Publisher interface in the Bridge.**

```go
// BAD: bridge/authorsvc/api.go
type EventPublisher interface {
    Publish(...) // Don't do this!
}
```

**Why?**
*   The Bridge defines the **Service API** (Inbound).
*   The Publisher is a **Dependency** (Outbound).
*   Putting outbound interfaces in the bridge couples consumers to the producer's infrastructure needs.
*   Keep the `EventPublisher` interface inside the **Producer's Application Layer** (`internal/application/ports`).
