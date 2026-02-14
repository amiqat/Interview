# Debugging & Performance

Scenario-based questions on diagnosing production issues and optimising performance in .NET applications.

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

### 2. 🟡 Your team's EF Core queries are causing performance issues on a table with 50 million rows. Walk through your investigation.

**Expected approach:**
1. **Capture the generated SQL** — `ToQueryString()` or SQL Profiler.
2. **Analyse execution plans** — Look for table scans, missing indexes, key lookups.
3. **Check for N+1** — Is lazy loading silently executing hundreds of queries?
4. **Evaluate the access pattern** — Do we need all columns? Use projections (`.Select()`). Do we need tracking? Use `AsNoTracking()`.
5. **Consider alternatives** — Split queries, raw SQL for complex reports, `SqlBulkCopy` for writes, read replicas for heavy reads.
6. **Index strategy** — Covering indexes for the most expensive queries, filtered indexes where applicable.

**Hint:** A principal-level candidate thinks about the problem holistically — not just "add an index" but also whether the data access pattern itself needs to change.

**🚩 Red Signal:** Adds `AsNoTracking()` to everything and calls it done, or has never looked at a SQL execution plan.
