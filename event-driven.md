# Event-Driven Architecture — Senior Interview Questions (Concepts Only)

## Core Concepts

### 1. What is event-driven architecture and when should you use it?

**Hint:** System components communicate by producing and consuming events (facts about something that happened). Use for decoupling services, async processing, real-time reactions, and scaling independently. Not ideal for simple CRUD or synchronous request-response flows.

**🚩 Red Signal:** Proposes event-driven architecture for simple CRUD apps or cannot explain when it's NOT appropriate.

---

### 2. What is the difference between an event, a command, and a query?

**Hint:** Event: something that happened (past tense, immutable, e.g., `OrderPlaced`). Command: a request to do something (imperative, e.g., `PlaceOrder`). Query: a request for data. Events are broadcast (pub/sub), commands are sent to a specific handler (point-to-point).

**🚩 Red Signal:** Confuses events with commands, or names events as commands (e.g., `CreateOrder` instead of `OrderCreated`).

---

### 3. Explain publish/subscribe vs point-to-point messaging.

**Hint:** Pub/Sub: publisher sends to a topic, multiple subscribers receive independently. Point-to-point: message sent to a queue, exactly one consumer processes it. Use pub/sub for events (broadcast), point-to-point for commands (single handler).

**🚩 Red Signal:** Cannot explain when to use a topic vs a queue, or conflates the two patterns.

---

## Patterns & Guarantees

### 4. What does "at-least-once" vs "at-most-once" vs "exactly-once" delivery mean?

**Hint:** At-least-once: message may be delivered multiple times (consumer must be idempotent). At-most-once: message may be lost but never duplicated. Exactly-once: extremely hard to achieve end-to-end; usually "effectively once" via idempotency + deduplication.

**🚩 Red Signal:** Assumes the message broker guarantees exactly-once delivery out of the box, or doesn't mention idempotency.

---

### 5. What is idempotency and why is it critical in event-driven systems?

**Hint:** Processing the same message multiple times produces the same result. Implement via idempotency keys, deduplication tables, or conditional writes. Essential because retries and at-least-once delivery can cause duplicate processing.

**🚩 Red Signal:** No strategy for handling duplicate messages, or assumes duplicates never happen.

---

### 6. What is eventual consistency and how do you deal with it?

**Hint:** After an event is published, other services will eventually reflect the change — but not immediately. Design UIs and APIs to tolerate stale reads. Use compensation/rollback for failures. Communicate SLAs for propagation time.

**🚩 Red Signal:** Designs systems expecting immediate consistency across services, or cannot explain how to handle the "eventually" part.

---

### 7. Explain the Outbox Pattern.

**Hint:** Write the event to an "outbox" table in the same database transaction as the business data. A separate process (poller or CDC) reads the outbox and publishes to the message broker. Guarantees atomicity between data change and event publication.

**🚩 Red Signal:** Publishes events directly to a broker inside a DB transaction (dual write problem), or doesn't know the outbox pattern.

---

### 8. What is the Saga pattern?

**Hint:** Manages a long-running business process across multiple services using a sequence of local transactions. Each step emits an event triggering the next. On failure, compensating transactions undo previous steps. Two styles: orchestration (central coordinator) and choreography (each service reacts).

**🚩 Red Signal:** Uses distributed transactions (2PC) instead of sagas for microservice workflows, or doesn't know how to handle compensation.

---

## Design Considerations

### 9. What is event sourcing and how does it differ from traditional CRUD?

**Hint:** Store all state changes as an immutable sequence of events (the event log IS the source of truth). Current state is derived by replaying events. Benefits: full audit trail, temporal queries, easy debugging. Drawbacks: complexity, eventual consistency, snapshot management.

**🚩 Red Signal:** Conflates event sourcing with event-driven architecture, or proposes it for every service without considering the complexity trade-off.

---

### 10. What is CQRS and how does it relate to event-driven architecture?

**Hint:** Command Query Responsibility Segregation — separate read and write models. Writes go through commands → events → write store. Reads query a denormalized read model (projection). Fits naturally with event sourcing and event-driven updates of the read model.

**🚩 Red Signal:** Implements CQRS in a simple app with one database or cannot explain when it adds value vs complexity.

---

### 11. How do you handle schema evolution for events?

**Hint:** Events are contracts — changing them can break consumers. Use versioning (e.g., `OrderPlacedV2`), optional fields, schema registries, or upcasters that transform old events to new format. Backward and forward compatibility (like Avro or Protobuf).

**🚩 Red Signal:** Changes event schemas without versioning or backward compatibility, breaking downstream consumers.

---

### 12. What are dead letter queues and why are they important?

**Hint:** Messages that repeatedly fail processing are moved to a DLQ instead of being retried forever. Prevents a single poison message from blocking the entire queue. Monitor DLQs, alert on growth, and have a process to inspect and replay or discard.

**🚩 Red Signal:** No strategy for failed messages — retries forever or silently drops messages without logging.
