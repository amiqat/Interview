# Event-Driven Architecture

Core concepts of event-driven systems — what they are, why they matter, and the key patterns every senior .NET developer should understand.

---

### 1. What is event-driven architecture and why would you choose it over synchronous request/response?

Components communicate through events — a producer publishes a fact ("something happened") and zero or more consumers react independently. The producer does not wait for or even know about the consumers.

**Why choose it:** Loose coupling between services, independent scaling of producers and consumers, resilience (events queue up if a consumer is temporarily down).

**Trade-offs:** Eventual consistency instead of immediate consistency, harder to trace and debug, requires infrastructure (message broker).

**Hint:** Look for a balanced answer that includes both benefits and real challenges. Principal-level thinking means they weigh trade-offs, not just list advantages.

**🚩 Red Signal:** Cannot name a single downside, or confuses async programming (`async`/`await`) with event-driven architecture.

---

### 2. What is the difference between an event and a command?

| | Event | Command |
|---|---|---|
| Semantics | Past tense — "OrderPlaced" | Imperative — "PlaceOrder" |
| Receivers | Many subscribers (pub/sub) | Exactly one handler (point-to-point) |
| Ownership | Producer owns the event | Consumer owns the command contract |
| Failure | Producer doesn't know/care if handling fails | Sender expects the command to be processed |

**Hint:** This is a fundamental concept. The candidate should explain *why* the distinction matters — it drives how messages are routed, how errors are handled, and how systems are coupled.

**🚩 Red Signal:** Uses "event" and "command" interchangeably or cannot explain when to use each.

---

### 3. What does "eventual consistency" mean and when is it acceptable?

After an event is published, different parts of the system may be temporarily out of sync. They will converge to a consistent state once all events are processed.

**Acceptable:** Notifications, search indexing, reporting, audit logs — where a delay of seconds or minutes is fine.

**Not acceptable:** Financial transactions that require immediate balance checks, inventory decrements where overselling is costly.

**Hint:** A strong answer includes strategies: idempotent consumers, compensating transactions, and UI patterns (optimistic updates, "processing…" states).

**🚩 Red Signal:** Claims all operations can be eventually consistent, or insists everything must be immediately consistent in a distributed system.

---

### 4. What is the Outbox Pattern and what problem does it solve?

**Problem (dual-write):** You save data to the database and then publish an event to a broker. If the publish fails after the save (or vice versa), the system is inconsistent.

**Solution:** Write the event to an "outbox" table in the **same database transaction** as the business data. A separate process reads the outbox and publishes to the broker. If it fails, it retries — the outbox is the source of truth.

**Hint:** The candidate should understand that this means events may be published more than once, so consumers must be idempotent.

**🚩 Red Signal:** Publishes events right after `SaveChanges()` and assumes the broker call always succeeds.

---

### 5. Explain the Saga pattern in simple terms. Choreography vs orchestration?

A Saga coordinates a multi-step business process across services without a distributed transaction. If any step fails, previous steps are undone via compensating actions.

**Choreography:** Each service reacts to events and publishes its own events. No central coordinator. Simple for 2–3 steps, confusing at scale.

**Orchestration:** A central "orchestrator" service tells each participant what to do and handles failures. Easier to understand, but the orchestrator is a single point of logic.

**Example:** Order → Payment → Inventory → Shipping. If inventory fails, the saga triggers a payment refund (compensating action).

**Hint:** Ask for a concrete example. A principal-level candidate should describe compensating actions for each step.

**🚩 Red Signal:** Suggests using distributed transactions (2PC) across microservices, or has no plan for partial failures.

---

### 6. How do you ensure a consumer handles the same event twice without causing problems (idempotency)?

Brokers guarantee **at-least-once** delivery, not exactly-once. The same event can arrive multiple times. Strategies:

- **Natural idempotency** — `UPDATE Status = 'Shipped' WHERE Id = @id` is safe to repeat.
- **Deduplication** — Store processed `MessageId`s; skip if already seen.
- **Conditional logic** — `UPDATE ... WHERE Status != 'Shipped'` prevents re-processing.

**Hint:** The deduplication check and business operation must be in the **same transaction** to avoid race conditions.

**🚩 Red Signal:** Assumes the broker delivers each message exactly once and takes no precautions.

---

### 7. What is a dead-letter queue and why do you need one?

When a message fails processing after a configured number of retries, it is moved to a dead-letter queue (DLQ) instead of being retried forever or silently dropped. This prevents a single "poison" message from blocking the queue.

**Workflow:** Retry with backoff → Exhaust retries → Move to DLQ → Alert the team → Investigate → Fix and replay.

**Hint:** The candidate should mention configuring retry counts, backoff delays, and monitoring/alerting on DLQ depth.

**🚩 Red Signal:** Has no plan for messages that repeatedly fail, or lets them block the queue indefinitely.

---

### 8. What are the main messaging patterns: pub/sub, point-to-point, and request/reply?

- **Pub/Sub:** One producer, many consumers. Each consumer gets a copy. Use for events. Example: `OrderPlaced` → email service, analytics service, inventory service all react.
- **Point-to-Point:** One producer, one consumer. Use for commands. Example: `SendEmailCommand` → email service.
- **Request/Reply:** Producer sends a message and waits for a response on a reply queue. Use sparingly — it reintroduces coupling.

**Hint:** The candidate should know which pattern to use for which scenario and why mixing them up causes architectural problems.

**🚩 Red Signal:** Uses pub/sub for everything, including commands that should have exactly one handler.

---

### 9. You need to build an order processing pipeline. Walk through the events and services involved.

Example flow:
1. **Order Service** → publishes `OrderPlaced`
2. **Payment Service** → consumes `OrderPlaced`, charges the customer, publishes `PaymentCompleted`
3. **Inventory Service** → consumes `PaymentCompleted`, reserves stock, publishes `StockReserved`
4. **Shipping Service** → consumes `StockReserved`, creates shipment, publishes `OrderShipped`
5. **Notification Service** → consumes `OrderShipped`, sends email/SMS

**If payment fails:** Payment Service publishes `PaymentFailed` → Order Service marks order as failed.
**If stock unavailable:** Inventory Service publishes `StockUnavailable` → Payment Service refunds → Order Service marks failed.

**Hint:** Look for clear compensating actions and awareness that each service owns its data and communicates only through events.

**🚩 Red Signal:** Describes a single monolithic service calling all steps sequentially, or has no failure/compensation strategy.

---

### 10. How do you choose between a message broker (RabbitMQ, Azure Service Bus) and an event streaming platform (Kafka)?

| | Message Broker | Event Streaming |
|---|---|---|
| Model | Messages consumed and removed | Events appended to a log, retained |
| Replay | Not supported natively | Consumers can replay from any offset |
| Ordering | Per-queue | Per-partition |
| Best for | Commands, transactional workflows | Event sourcing, analytics, audit logs |

**Hint:** A principal-level candidate understands that the choice depends on the use case — Kafka for high-throughput event streaming and replay; RabbitMQ/Service Bus for transactional messaging with routing and DLQ.

**🚩 Red Signal:** Picks a technology without understanding the fundamental model difference, or always defaults to one tool regardless of requirements.
