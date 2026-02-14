# System Design

Scenario-based questions on designing and scaling .NET systems for real-world requirements.

---

### 1. 🔴 You are designing a multi-tenant SaaS application. How do you isolate tenant data?

**Expected approaches (with trade-offs):**

| Strategy | Isolation | Complexity | Cost |
|---|---|---|---|
| Separate database per tenant | Strongest | High (connection management, migrations) | High |
| Shared database, separate schema | Good | Medium | Medium |
| Shared database, shared schema with `TenantId` column | Weakest | Low | Low |

**Implementation with shared schema:**
- EF Core Global Query Filters: `builder.Entity<Order>().HasQueryFilter(o => o.TenantId == currentTenantId)`.
- Resolve `currentTenantId` from JWT claims, request header, or subdomain.
- Row-Level Security in SQL Server as an additional safety net.

**Hint:** A principal-level candidate discusses the trade-offs and picks a strategy based on requirements (compliance, scale, cost). They also mention the risk of a developer forgetting the filter and leaking data across tenants.

**🚩 Red Signal:** No awareness of data isolation concerns, or uses a single `TenantId` column with no safeguards (no global filter, no RLS).

---

### 2. 🟡 You need to implement real-time notifications (e.g., order status updates) for a web application. What approach do you choose and why?

**Options:**
- **SignalR** — Bi-directional communication over WebSockets (with fallbacks). Best for .NET ecosystems. Supports groups (send to a user, a role, all clients).
- **Server-Sent Events (SSE)** — Unidirectional (server → client). Simpler, HTTP-based, works through most proxies. Good when client doesn't need to send messages.
- **Polling** — Simplest but least efficient. Acceptable for low-frequency updates.

**Scaling SignalR:** Use a backplane (Redis, Azure SignalR Service) so messages reach clients connected to any server instance.

**Hint:** The answer should be driven by requirements: frequency of updates, number of concurrent clients, infrastructure constraints. A principal-level developer doesn't just pick "WebSockets because they're the best."

**🚩 Red Signal:** Only knows polling, or picks WebSockets without considering scaling implications.

---

### 3. 🔴 You need to design a high-throughput API that ingests millions of events per day. What architectural decisions do you make?

**Expected approach:**
1. **Receive fast, process later** — Accept the event, write to a queue (Kafka, Service Bus, SQS), and return `202 Accepted`. Process asynchronously.
2. **Horizontal scaling** — Stateless API behind a load balancer. Consumer instances scale based on queue depth.
3. **Storage** — Time-series or append-only store for events. Consider partitioning by tenant or date.
4. **Backpressure** — If consumers fall behind, the queue absorbs the burst. Monitor queue depth and auto-scale consumers.
5. **Idempotency** — Producers may retry, so consumers must handle duplicates.
6. **Observability** — Track ingestion rate, processing latency (publish → consumed), error rate, and queue depth.

**Hint:** Look for the "accept and queue" pattern rather than synchronous processing. A principal-level candidate designs for failure modes (what if the queue is full? what if a consumer crashes mid-processing?).

**🚩 Red Signal:** Processes each event synchronously in the API request, or has no strategy for handling spikes.
