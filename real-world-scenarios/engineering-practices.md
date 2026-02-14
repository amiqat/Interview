# Engineering Practices

Scenario-based questions on migration, resilience, incident response, CI/CD, and testing strategies.

---

### 1. 🔴 You need to migrate a monolithic .NET Framework 4.8 application to .NET 8. How do you plan the migration?

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

### 2. 🟡 A third-party payment API is intermittently failing (timeouts, 500 errors). How do you make your system resilient to this?

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

### 3. 🟡 A critical production bug is affecting 10% of users. Walk through your incident response.

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

### 4. 🟡 Your CI/CD pipeline takes 45 minutes. How do you reduce it?

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

### 5. 🟡 You are joining a team with no unit tests and a tightly coupled codebase. How do you introduce testing?

**Expected approach:**
1. **Don't try to test everything at once** — Start with high-value targets: critical business logic, frequently-broken code, and new features.
2. **Characterisation tests** — Write tests that document current behaviour before refactoring.
3. **Seam-based refactoring** — Introduce interfaces at boundaries (database, external APIs) to enable mocking. Do this incrementally, not as a big-bang refactor.
4. **Integration tests first** — Often easier in a tightly coupled codebase; test at the API level with `WebApplicationFactory<T>`.
5. **Team buy-in** — Pair program on tests, include test coverage in PR reviews, celebrate improvements.
6. **Test pyramid** — Work toward: many unit tests, fewer integration tests, very few end-to-end tests.

**Hint:** A principal-level engineer shows pragmatism and leadership — they know that mandating "100% coverage" on a legacy codebase will fail. They lead by example.

**🚩 Red Signal:** Demands immediate 80% code coverage or says "we don't need tests, we have QA."
