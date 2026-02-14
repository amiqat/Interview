# Event-Driven Patterns

Patterns for reliable messaging, failure handling, and technology selection in event-driven systems.

---

### 1. 🟢 What does "idempotent" mean in the context of message processing?

Processing the same message twice produces the same result — no duplicate records, no double charges, no side effects beyond the first processing. This is essential because message brokers guarantee at-least-once delivery, not exactly-once.

**Hint:** The candidate should give a concrete example, such as using a unique message ID to skip already-processed messages.

**🚩 Red Signal:** Assumes brokers deliver each message exactly once and takes no precautions for duplicates.

---

### 2. 🟡 You save an order to the database and then publish an `OrderPlaced` event to the broker. The publish fails after the DB commit, leaving the system inconsistent. How do you solve this?

**Problem (dual-write):** You save data to the database and then publish an event to a broker. If the publish fails after the save (or vice versa), the system is inconsistent.

**Solution (Outbox Pattern):** Write the event to an "outbox" table in the **same database transaction** as the business data. A separate process (relay) reads the outbox and publishes to the broker. If the relay fails, it retries — the outbox is the source of truth.

This means events may be published more than once, so consumers must be idempotent. The outbox relay can use polling or change data capture (CDC) to detect new messages.

**Hint:** The candidate should understand that this trades the dual-write problem for at-least-once delivery, which requires idempotent consumers.

**🚩 Red Signal:** Publishes events right after `SaveChanges()` and assumes the broker call always succeeds.

---

### 3. 🟡 Your order processing flow spans four services: Order → Payment → Inventory → Shipping. The inventory service fails after payment has already been charged. How do you undo the payment?

A Saga coordinates a multi-step business process across services without a distributed transaction. If any step fails, previous steps are undone via **compensating actions**.

**Choreography:** Each service reacts to events and publishes its own events. No central coordinator. Simple for 2–3 steps, confusing at scale.

**Orchestration:** A central "orchestrator" service tells each participant what to do and handles failures. Easier to understand, but the orchestrator is a single point of logic.

**In this scenario:** When Inventory Service publishes `StockUnavailable`, the saga triggers a compensating action — Payment Service refunds the charge, and Order Service marks the order as failed.

**Hint:** Ask for a concrete example. A principal-level candidate should describe compensating actions for each step.

**🚩 Red Signal:** Suggests using distributed transactions (2PC) across microservices, or has no plan for partial failures.

---

### 4. 🟡 Your message broker delivers the same `OrderPlaced` event twice. Your consumer inserts a new order record each time, creating a duplicate. How do you prevent this?

Brokers guarantee **at-least-once** delivery, not exactly-once. The same event can arrive multiple times. Strategies:

- **Natural idempotency** — `UPDATE Status = 'Shipped' WHERE Id = @id` is safe to repeat.
- **Deduplication** — Store processed `MessageId`s in a deduplication table; skip if already seen.
- **Conditional logic** — `INSERT ... WHERE NOT EXISTS (SELECT 1 FROM Orders WHERE OrderId = @id)` prevents duplicate inserts.

The deduplication check and business operation must be in the **same transaction** to avoid race conditions.

**Hint:** Look for understanding that idempotency is not optional in event-driven systems — it's a requirement driven by broker delivery guarantees.

**🚩 Red Signal:** Assumes the broker delivers each message exactly once and takes no precautions.

---

### 5. 🔴 Design an order processing pipeline end-to-end. Walk through the events, services involved, and how you handle failures at each step.

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

### 6. 🟡 Your team is choosing between RabbitMQ (or Azure Service Bus) and Kafka for a new system. What factors drive this decision?

| | Message Broker (RabbitMQ, Service Bus) | Event Streaming (Kafka) |
|---|---|---|
| Model | Messages consumed and removed | Events appended to a log, retained |
| Replay | Not supported natively | Consumers can replay from any offset |
| Ordering | Per-queue | Per-partition |
| Best for | Commands, transactional workflows | Event sourcing, analytics, audit logs |

Choose a **message broker** when you need transactional messaging with routing, dead-letter queues, and per-message acknowledgment. Choose **Kafka** when you need high-throughput event streaming, replay capability, and multiple consumer groups reading the same data independently.

**Hint:** A principal-level candidate understands that the choice depends on the use case — Kafka for high-throughput event streaming and replay; RabbitMQ/Service Bus for transactional messaging with routing and DLQ.

**🚩 Red Signal:** Picks a technology without understanding the fundamental model difference, or always defaults to one tool regardless of requirements.
