# Event-Driven Architecture

Questions on event-driven concepts, messaging patterns, and their application in .NET systems.

---

### 1. What is event-driven architecture and how does it differ from request-driven (synchronous) architecture?

In event-driven architecture, components communicate by producing and consuming events (asynchronous messages). Producers do not wait for consumers to process the event — they fire and forget. In request-driven architecture, the caller waits for a response (HTTP request/response, RPC).

Key benefits: loose coupling, scalability (consumers can scale independently), resilience (events can be queued during downstream outages).

**Hint:** A strong candidate discusses the trade-offs: eventual consistency, event ordering challenges, and the difficulty of debugging distributed async flows.

**🚩 Red Signal:** Cannot articulate the difference between synchronous and asynchronous communication patterns, or sees no downsides to event-driven architecture.

---

### 2. Explain the difference between events, commands, and messages in a messaging system.

- **Event:** A notification that something happened (past tense): `OrderPlaced`, `PaymentReceived`. Published to multiple subscribers. The producer does not know or care who handles it.
- **Command:** A request to do something (imperative): `PlaceOrder`, `SendEmail`. Sent to a single handler. The sender expects it to be processed.
- **Message:** The generic envelope that carries either an event or a command through the transport.

**Hint:** In MassTransit/NServiceBus, events use publish/subscribe; commands use send to a specific queue. The candidate should understand why this distinction matters for routing and error handling.

**🚩 Red Signal:** Uses the terms interchangeably without understanding the semantic and routing differences.

---

### 3. What is the Outbox Pattern and why is it important for reliable messaging?

The Outbox Pattern ensures that a domain operation (e.g., saving an order) and its corresponding event publication are atomic. Instead of publishing directly to a message broker, the event is written to an outbox table in the same database transaction. A background process reads the outbox and publishes events to the broker.

This prevents the dual-write problem: if the database commit succeeds but the broker publish fails (or vice versa), the system ends up in an inconsistent state.

**Hint:** Look for awareness of idempotent consumers (events may be delivered more than once when the outbox resends), and tools like MassTransit's transactional outbox or NServiceBus Outbox.

**🚩 Red Signal:** Publishes events directly after `SaveChanges()` without considering what happens if the publish fails.

---

### 4. How do you implement the pub/sub pattern in .NET?

Options:
- **In-process:** MediatR notifications, C# events/delegates, `System.Threading.Channels`.
- **Distributed:** RabbitMQ, Azure Service Bus, Kafka with libraries like MassTransit or NServiceBus.
- **Cloud-native:** Azure Event Grid, AWS EventBridge.

In MassTransit: define an `IConsumer<OrderPlaced>`, register it with the bus, and publish via `IBus.Publish(new OrderPlaced { ... })`.

**Hint:** The candidate should distinguish between in-process events (same deployment unit) and distributed events (across services), and explain why distributed events need a durable broker.

**🚩 Red Signal:** Uses in-memory events for cross-service communication in a distributed system.

---

### 5. What is eventual consistency and how do you design systems that tolerate it?

In an event-driven system, after an event is published, consumers may take time to process it — the system is not immediately consistent but will become consistent eventually.

Design strategies:
- **Idempotent consumers** — processing the same event twice produces the same result.
- **Correlation IDs** — trace a business process across multiple services.
- **Compensating transactions** — undo partial operations if a downstream step fails (Saga pattern).
- **UI patterns** — optimistic updates, showing "processing" states to the user.

**Hint:** A senior developer should discuss when eventual consistency is acceptable (notifications, reporting) vs when strong consistency is required (payment processing, inventory decrements).

**🚩 Red Signal:** Assumes all operations in a distributed system can be strongly consistent, or does not understand what eventual consistency means.

---

### 6. Explain the Saga pattern. How does it manage distributed transactions?

A Saga is a sequence of local transactions across multiple services. Each step either succeeds and triggers the next, or fails and triggers compensating actions for all previously completed steps. There are two coordination styles:

- **Choreography:** Each service listens for events and decides what to do next. Simple but hard to understand at scale.
- **Orchestration:** A central orchestrator directs each step. Easier to reason about but introduces a coordinator dependency.

**Hint:** The candidate should give a concrete example (e.g., order → payment → inventory → shipping) and explain what the compensating action is for each step (e.g., refund payment if inventory reservation fails).

**🚩 Red Signal:** Tries to use distributed transactions (2PC/DTC) across microservices, or has no strategy for handling partial failures.

---

### 7. How does MassTransit (or a similar library) simplify building event-driven .NET applications?

MassTransit provides:
- **Transport abstraction** — swap between RabbitMQ, Azure Service Bus, Amazon SQS, and in-memory for testing.
- **Consumer pattern** — implement `IConsumer<T>` with automatic message deserialization and DI integration.
- **Retry, circuit breaker, and dead-letter** — built-in resilience policies.
- **Saga state machines** — `MassTransitStateMachine<T>` for orchestration.
- **Transactional outbox** — reliable event publishing with EF Core integration.

**Hint:** Look for experience with at least one messaging library. The candidate should explain how consumers are registered, how message topologies are configured, and how testing works (using the in-memory transport).

**🚩 Red Signal:** Has built an event-driven system but uses hand-rolled `BackgroundService` consumers with no retry, no dead-letter, and no observability.

---

### 8. What are dead-letter queues and why are they important?

A dead-letter queue (DLQ) is where messages are moved after they fail processing a configured number of times. This prevents a single poison message from blocking the entire queue and provides a place for manual inspection and replay.

**Hint:** The candidate should explain the retry → move to DLQ → alert → investigate → replay workflow, and how to configure retry counts and backoff strategies.

**🚩 Red Signal:** Has no strategy for handling messages that repeatedly fail, or lets failed messages silently disappear.

---

### 9. How do you ensure idempotency in event consumers?

Strategies:
- **Natural idempotency** — the operation is inherently idempotent (e.g., setting a status to "Completed").
- **Deduplication table** — store processed `MessageId`s and skip duplicates.
- **Conditional writes** — `UPDATE ... WHERE Status != 'Completed'` or optimistic concurrency.
- **Idempotency key** — passed by the producer and checked before processing.

**Hint:** The deduplication table should be checked and updated within the same transaction as the business operation to avoid race conditions.

**🚩 Red Signal:** Assumes messages are delivered exactly once ("the broker guarantees it") without implementing any idempotency safeguards.

---

### 10. How do you test event-driven systems?

- **Unit tests:** Test consumer logic in isolation using MassTransit's `InMemoryTestHarness` or by directly invoking the consumer with a mock context.
- **Integration tests:** Use the in-memory transport (MassTransit) or Testcontainers (RabbitMQ, Kafka) to verify end-to-end message flow.
- **Contract tests:** Ensure producer and consumer agree on event schema (e.g., using a schema registry or shared contracts NuGet package).
- **Observability in production:** Correlation IDs, distributed tracing (OpenTelemetry), and structured logging.

**Hint:** Testing async flows is hard. Look for strategies like polling/waiting for expected state changes, and awareness that timing-based assertions are fragile.

**🚩 Red Signal:** Has no testing strategy for event-driven components or tests only the happy path synchronously.
