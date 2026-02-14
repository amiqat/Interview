# Real-World Scenarios

Cross-cutting scenarios that test how candidates apply their knowledge to practical problems. These questions have no single correct answer — evaluate the thinking process, trade-off analysis, and communication clarity.

---

### 1. 🔴 A production API endpoint that previously responded in 50ms now takes 5 seconds. How do you diagnose and fix this?

**Expected approach:**
1. **Reproduce** — Is it consistent or intermittent? All endpoints or just one?
2. **Check infrastructure** — CPU, memory, disk I/O, network. Is the database server healthy?
3. **Application-level tracing** — Look at logs, distributed traces (OpenTelemetry), or APM (Application Insights).
4. **Database** — Check for query plan regressions (Query Store), missing indexes, parameter sniffing, lock contention.
5. **Thread Pool** — Check for thread starvation (`dotnet-counters` → ThreadPool queue length). Look for synchronous blocking in async code.
6. **External dependencies** — Third-party API slow? Circuit breaker tripped?

**Hint:** A principal-level candidate has a structured diagnostic approach — they don't guess randomly. Look for familiarity with specific tools: `dotnet-counters`, `dotnet-trace`, PerfView, Application Insights, SQL Profiler.

**🚩 Red Signal:** Immediately jumps to "restart the server" or "add more RAM" without any diagnostic process.

---

### 2. 🔴 You need to migrate a monolithic .NET Framework 4.8 application to .NET 8. How do you plan the migration?

**Expected approach:**
1. **Inventory** — List all dependencies, NuGet packages, and .NET Framework-specific APIs (WCF, ASMX, `System.Web`).
2. **Use .NET Upgrade Assistant** — Automated tooling for initial conversion.
3. **Strangler Fig pattern** — Migrate incrementally: new features in .NET 8, route traffic gradually, keep the monolith running in parallel.
4. **Shared contracts** — Use API gateways or messaging to bridge old and new services during migration.
5. **Test extensively** — Maintain integration tests that validate behaviour before and after.
6. **Replace incompatible APIs** — WCF → gRPC or REST, `System.Web` → ASP.NET Core middleware.

**Hint:** A senior candidate talks about risk mitigation, not a big-bang rewrite. Look for awareness of the Strangler Fig pattern and incremental delivery.

**🚩 Red Signal:** Proposes rewriting everything from scratch in a single sprint, or has never migrated a codebase.

---

### 3. 🟡 Your team's EF Core queries are causing performance issues on a table with 50 million rows. Walk through your investigation.

**Expected approach:**
1. **Capture the generated SQL** — `ToQueryString()` or SQL Profiler.
2. **Analyse execution plans** — Look for table scans, missing indexes, key lookups.
3. **Check for N+1** — Is lazy loading silently executing hundreds of queries?
4. **Evaluate the access pattern** — Do we need all columns? Use projections (`.Select()`). Do we need tracking? Use `AsNoTracking()`.
5. **Consider alternatives** — Split queries, raw SQL for complex reports, `SqlBulkCopy` for writes, read replicas for heavy reads.
6. **Index strategy** — Covering indexes for the most expensive queries, filtered indexes where applicable.

**Hint:** A principal-level candidate thinks about the problem holistically — not just "add an index" but also whether the data access pattern itself needs to change.

**🚩 Red Signal:** Adds `AsNoTracking()` to everything and calls it done, or has never looked at a SQL execution plan.

---

### 4. 🟡 A third-party payment API is intermittently failing (timeouts, 500 errors). How do you make your system resilient to this?

**Expected approach:**
1. **Retry with backoff** — Use Polly (or `Microsoft.Extensions.Http.Resilience` in .NET 8) for transient failures. Exponential backoff + jitter.
2. **Circuit breaker** — After N failures, stop calling the API for a cooldown period. Return a graceful fallback.
3. **Timeout policy** — Set an explicit `HttpClient` timeout shorter than the default 100s.
4. **Queue-based approach** — Enqueue payment requests and process them asynchronously. Retry from the queue.
5. **Idempotency** — Ensure retries don't charge the customer twice (idempotency keys).
6. **Monitoring** — Alert on error rate, circuit breaker state changes, and retry counts.

**Hint:** Look for a layered resilience strategy, not just "retry 3 times." Principal-level thinking includes graceful degradation and user experience during outages.

**🚩 Red Signal:** Wraps the call in a try/catch, swallows the exception, and moves on — or retries infinitely without backoff.

---

### 5. 🔴 You are designing a multi-tenant SaaS application. How do you isolate tenant data?

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

### 6. 🟡 A critical production bug is affecting 10% of users. Walk through your incident response.

**Expected approach:**
1. **Assess severity** — How many users affected? Is data corrupted? Is revenue impacted?
2. **Communicate** — Notify stakeholders, post in incident channel, assign an incident lead.
3. **Contain** — Can we feature-flag the broken feature off? Roll back the last deployment?
4. **Diagnose** — Check recent deployments, review logs around the time reports started, trace affected requests.
5. **Fix** — Hotfix or rollback. If hotfix, test in staging first even under pressure.
6. **Post-mortem** — Blameless review: timeline, root cause, what detection failed, action items.

**Hint:** Assess the candidate's composure and structure under pressure. A principal-level engineer has a repeatable incident response process.

**🚩 Red Signal:** Panics and starts changing production code without understanding the root cause, or has never dealt with a production incident.

---

### 7. 🟡 You need to implement real-time notifications (e.g., order status updates) for a web application. What approach do you choose?

**Options:**
- **SignalR** — Bi-directional communication over WebSockets (with fallbacks). Best for .NET ecosystems. Supports groups (send to a user, a role, all clients).
- **Server-Sent Events (SSE)** — Unidirectional (server → client). Simpler, HTTP-based, works through most proxies. Good when client doesn't need to send messages.
- **Polling** — Simplest but least efficient. Acceptable for low-frequency updates.

**Scaling SignalR:** Use a backplane (Redis, Azure SignalR Service) so messages reach clients connected to any server instance.

**Hint:** The answer should be driven by requirements: frequency of updates, number of concurrent clients, infrastructure constraints. A principal-level developer doesn't just pick "WebSockets because they're the best."

**🚩 Red Signal:** Only knows polling, or picks WebSockets without considering scaling implications.

---

### 8. 🟡 Your CI/CD pipeline takes 45 minutes. How do you reduce it?

**Expected approach:**
1. **Profile** — Which stages take the longest? Build? Tests? Deployment?
2. **Parallelise tests** — Split integration and unit tests into parallel jobs.
3. **Cache dependencies** — NuGet restore, Docker layers, node_modules.
4. **Optimise Docker builds** — Multi-stage builds, layer caching, smaller base images.
5. **Test smarter** — Run only tests affected by changed files (test impact analysis). Keep slow integration tests in a separate pipeline.
6. **Build smarter** — Incremental builds, build only affected projects in a monorepo.

**Hint:** A principal-level candidate thinks about developer experience and feedback loops. A 45-minute pipeline means developers batch changes and get slow feedback.

**🚩 Red Signal:** Has never thought about pipeline performance, or suggests removing tests to make it faster.

---

### 9. 🟡 You are joining a team with no unit tests and a tightly coupled codebase. How do you introduce testing?

**Expected approach:**
1. **Don't try to test everything at once** — Start with high-value targets: critical business logic, frequently-broken code, and new features.
2. **Characterisation tests** — Write tests that document current behaviour before refactoring.
3. **Seam-based refactoring** — Introduce interfaces at boundaries (database, external APIs) to enable mocking. Do this incrementally, not as a big-bang refactor.
4. **Integration tests first** — Often easier in a tightly coupled codebase; test at the API level with `WebApplicationFactory<T>`.
5. **Team buy-in** — Pair program on tests, include test coverage in PR reviews, celebrate improvements.
6. **Test pyramid** — Work toward: many unit tests, fewer integration tests, very few end-to-end tests.

**Hint:** A principal-level engineer shows pragmatism and leadership — they know that mandating "100% coverage" on a legacy codebase will fail. They lead by example.

**🚩 Red Signal:** Demands immediate 80% code coverage or says "we don't need tests, we have QA."

---

### 10. 🔴 Design a high-throughput API that ingests millions of events per day. What architectural decisions do you make?

**Expected approach:**
1. **Receive fast, process later** — Accept the event, write to a queue (Kafka, Service Bus, SQS), and return `202 Accepted`. Process asynchronously.
2. **Horizontal scaling** — Stateless API behind a load balancer. Consumer instances scale based on queue depth.
3. **Storage** — Time-series or append-only store for events. Consider partitioning by tenant or date.
4. **Backpressure** — If consumers fall behind, the queue absorbs the burst. Monitor queue depth and auto-scale consumers.
5. **Idempotency** — Producers may retry, so consumers must handle duplicates.
6. **Observability** — Track ingestion rate, processing latency (publish → consumed), error rate, and queue depth.

**Hint:** Look for the "accept and queue" pattern rather than synchronous processing. A principal-level candidate designs for failure modes (what if the queue is full? what if a consumer crashes mid-processing?).

**🚩 Red Signal:** Processes each event synchronously in the API request, or has no strategy for handling spikes.
