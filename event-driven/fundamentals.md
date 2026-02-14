# Event-Driven Fundamentals

Core concepts of event-driven systems — what they are, when to use them, and the foundational building blocks every senior .NET developer should understand.

---

### 1. 🟢 Your e-commerce platform processes orders synchronously — validate, charge payment, reserve inventory, send email — all in one HTTP request taking 3 seconds. How would you redesign this with an event-driven approach, and what trade-offs would you accept?

Instead of doing everything synchronously, the API validates the order and publishes an `OrderPlaced` event, returning immediately (or with `202 Accepted`). Independent consumers handle payment, inventory, and email in parallel or in sequence. The producer does not wait for or even know about the consumers.

**Benefits:** Loose coupling between services, independent scaling of producers and consumers, resilience (events queue up if a consumer is temporarily down), much faster API response time.

**Trade-offs:** Eventual consistency instead of immediate consistency, harder to trace and debug end-to-end, requires infrastructure (message broker), and the user doesn't get instant confirmation of all steps.

**Hint:** Look for a balanced answer that includes both benefits and real challenges. Principal-level thinking means they weigh trade-offs, not just list advantages.

**🚩 Red Signal:** Cannot name a single downside, or confuses async programming (`async`/`await`) with event-driven architecture.

---

### 2. 🟢 A developer names a message `UpdateInventory` and publishes it to a topic with multiple subscribers. Another developer names their message `InventoryUpdated` and sends it to a single queue. Explain why the naming and routing choices matter.

| | Event | Command |
|---|---|---|
| Semantics | Past tense — "InventoryUpdated" | Imperative — "UpdateInventory" |
| Receivers | Many subscribers (pub/sub) | Exactly one handler (point-to-point) |
| Ownership | Producer owns the event | Consumer owns the command contract |
| Failure | Producer doesn't know/care if handling fails | Sender expects the command to be processed |

The first developer has it backwards: `UpdateInventory` is a command (imperative) and should go to a single queue, not a topic. The second developer is correct: `InventoryUpdated` is an event (past tense, fact) suitable for pub/sub. Mixing these up leads to misrouted messages, unclear ownership, and brittle coupling.

**Hint:** This is a fundamental concept. The candidate should explain *why* the distinction matters — it drives how messages are routed, how errors are handled, and how systems are coupled.

**🚩 Red Signal:** Uses "event" and "command" interchangeably or cannot explain when to use each.

---

### 3. 🟡 After publishing an `OrderPlaced` event, the search index and email service are temporarily out of sync — users see stale search results for a few seconds. Is this acceptable? When would it not be?

After an event is published, different parts of the system may be temporarily out of sync. They will converge to a consistent state once all events are processed. This is eventual consistency.

**Acceptable:** Notifications, search indexing, reporting, audit logs — where a delay of seconds or minutes is fine. In this scenario, stale search results for a few seconds is generally acceptable.

**Not acceptable:** Financial transactions that require immediate balance checks, inventory decrements where overselling is costly, or any operation where showing stale data could cause a user to take an irreversible incorrect action.

**Hint:** A strong answer includes strategies for managing eventual consistency: idempotent consumers, compensating transactions, and UI patterns (optimistic updates, "processing…" states).

**🚩 Red Signal:** Claims all operations can be eventually consistent, or insists everything must be immediately consistent in a distributed system.

---

### 4. 🟢 A consumer keeps crashing on a specific malformed message. The message gets redelivered endlessly, blocking all other messages in the queue. How do you prevent this?

When a message fails processing after a configured number of retries, it should be moved to a dead-letter queue (DLQ) instead of being retried forever or silently dropped. This prevents a single "poison" message from blocking the entire queue.

**Workflow:** Retry with exponential backoff → Exhaust retries → Move to DLQ → Alert the team → Investigate → Fix and replay.

The DLQ provides a holding area where failed messages can be inspected, the root cause fixed, and messages replayed once the fix is deployed.

**Hint:** The candidate should mention configuring retry counts, backoff delays, and monitoring/alerting on DLQ depth.

**🚩 Red Signal:** Has no plan for messages that repeatedly fail, or lets them block the queue indefinitely.

---

### 5. 🟢 You have three situations: (1) an order is placed and multiple services need to know, (2) you need to send one specific email, (3) you need to call a service and wait for a response. Which messaging pattern fits each and why?

- **Pub/Sub (situation 1):** One producer, many consumers. Each consumer gets a copy. Use for events. Example: `OrderPlaced` → email service, analytics service, inventory service all react independently.
- **Point-to-Point (situation 2):** One producer, one consumer. Use for commands. Example: `SendEmailCommand` → email service is the only handler.
- **Request/Reply (situation 3):** Producer sends a message and waits for a response on a reply queue. Use sparingly — it reintroduces temporal coupling.

**Hint:** The candidate should know which pattern to use for which scenario and why mixing them up causes architectural problems.

**🚩 Red Signal:** Uses pub/sub for everything, including commands that should have exactly one handler.
