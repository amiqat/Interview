# EF Core & LINQ

Questions on Entity Framework Core internals, query translation, change tracking, performance pitfalls, and advanced LINQ usage.

---

### 1. 🟡 How does EF Core translate LINQ queries to SQL? What happens when a LINQ expression cannot be translated?

EF Core's query pipeline parses the LINQ expression tree, converts it to a `SqlExpression` tree, and then generates parameterised SQL. If a part of the expression cannot be translated to SQL, EF Core 5+ throws an `InvalidOperationException` by default (client evaluation was silently done in earlier versions, causing N+1 problems).

**Hint:** A strong candidate mentions the `IQueryable` vs `IEnumerable` boundary, the danger of accidental client evaluation, and how to use `ToQueryString()` to inspect generated SQL during development.

**🚩 Red Signal:** Believes all C# lambda expressions are automatically translated to SQL, or is unaware that client evaluation was a major performance trap in EF Core 2.x.

---

### 2. 🟢 Explain the difference between `AsNoTracking()` and the default tracking behaviour in EF Core.

By default, EF Core tracks all entities returned by queries in its change tracker (an identity map). `AsNoTracking()` disables this, which is faster for read-only scenarios because it skips snapshot creation and identity resolution.

**Hint:** Bonus points for mentioning `AsNoTrackingWithIdentityResolution()` (keeps identity resolution without snapshot overhead) and `QueryTrackingBehavior.NoTracking` at the `DbContext` level.

**🚩 Red Signal:** Cannot explain what the change tracker does or why tracking has a memory/CPU cost.

---

### 3. 🟡 How do you handle bulk inserts efficiently in EF Core?

`SaveChanges()` sends one INSERT per entity by default — slow for thousands of rows. Solutions include:
- **EF Core 7+ `ExecuteUpdate`/`ExecuteDelete`** for set-based operations.
- **Third-party libraries** like `EFCore.BulkExtensions` that use `SqlBulkCopy` under the hood.
- **Direct `SqlBulkCopy`** via the underlying `SqlConnection` for maximum throughput.
- **Batching with `SaveChanges`** after every N entities to control memory.

**Hint:** Look for awareness that `AutoDetectChangesEnabled = false` and `Add` vs `AddRange` affect change-tracker performance during bulk loads.

**🚩 Red Signal:** Suggests calling `SaveChanges()` inside a tight loop for each entity, or is unaware of `SqlBulkCopy`.

---

### 4. 🟢 What is the N+1 query problem in EF Core and how do you solve it?

N+1 occurs when a query loads a collection of parent entities (1 query) and then lazily loads related children one-by-one (N queries). Solutions:
- **Eager loading** with `.Include()` / `.ThenInclude()`.
- **Explicit loading** via `context.Entry(entity).Collection(...).LoadAsync()`.
- **Split queries** via `.AsSplitQuery()` (EF Core 5+) to avoid cartesian explosion.
- **Projection** with `.Select()` to fetch only needed data.

**Hint:** A great candidate discusses the trade-off between a single large JOIN (cartesian explosion with multiple collections) vs split queries (multiple round-trips).

**🚩 Red Signal:** Relies entirely on lazy loading without understanding the performance implications.

---

### 5. 🔴 Explain compiled queries in EF Core. When should you use them?

`EF.CompileQuery` / `EF.CompileAsyncQuery` pre-compiles the LINQ expression tree and caches the query plan, removing the overhead of expression tree processing on every call. Useful for hot-path queries that are called thousands of times per second.

**Hint:** In EF Core 6+ the query cache is already effective for most cases; compiled queries help only when profiling shows LINQ compilation is a bottleneck.

**🚩 Red Signal:** Cannot explain the difference between query compilation time and database execution time.

---

### 6. 🟡 How do EF Core migrations work and what strategies exist for deploying them in production?

Migrations generate C# code that represents schema changes. Application strategies:
- **`dotnet ef database update`** — good for development, risky for production.
- **`context.Database.MigrateAsync()`** at startup — simple but problematic in scaled-out deployments (race conditions).
- **Generating SQL scripts** (`dotnet ef migrations script --idempotent`) and applying them via CI/CD pipelines — safest for production.

**Hint:** Look for awareness of idempotent scripts, migration bundles (EF Core 6+), and the `__EFMigrationsHistory` table.

**🚩 Red Signal:** Applies migrations manually in production without a scripted, repeatable process.

---

### 7. 🟢 What is the difference between `IQueryable<T>` and `IEnumerable<T>` in the context of EF Core?

`IQueryable<T>` builds an expression tree that is translated to SQL and executed on the database server. `IEnumerable<T>` executes in-memory in the application. Casting an `IQueryable` to `IEnumerable` (e.g., by calling `.ToList()` too early or using incompatible LINQ methods) forces client-side evaluation of subsequent operations.

**Hint:** Candidates should know that `Where()` on `IQueryable<T>` adds a SQL `WHERE` clause, but `Where()` on `IEnumerable<T>` filters in memory after fetching all rows.

**🚩 Red Signal:** Uses `.ToList()` immediately after `DbSet` and then applies further LINQ filtering in memory on large datasets.

---

### 8. 🟡 How do you use raw SQL in EF Core without losing the benefits of the ORM?

- **`FromSqlRaw` / `FromSqlInterpolated`** — executes raw SQL and maps results to entities; can be composed with further LINQ (`.Where()`, `.OrderBy()`).
- **`ExecuteSqlRaw` / `ExecuteSqlInterpolated`** — for non-query commands (INSERT/UPDATE/DELETE).
- **`SqlQuery<T>`** (EF Core 7+) — maps raw SQL to arbitrary types, not just entities.

**Hint:** Look for emphasis on parameterised queries to prevent SQL injection. `FromSqlInterpolated` uses `FormattableString` for safe parameter binding.

**🚩 Red Signal:** Concatenates user input directly into raw SQL strings.

---

### 9. 🟡 How does EF Core handle concurrency conflicts?

EF Core supports optimistic concurrency via a concurrency token (a `[ConcurrencyCheck]` column or a `[Timestamp]`/`rowversion` column). When `SaveChanges()` detects that the token value in the database differs from the tracked value, it throws `DbUpdateConcurrencyException`.

**Hint:** The candidate should explain the resolution strategy: reload the entity, merge changes, and retry — potentially with a loop. Awareness of `entry.OriginalValues`, `entry.CurrentValues`, and `entry.GetDatabaseValues()`.

**🚩 Red Signal:** Has no strategy for handling concurrent updates or relies solely on database-level locking for all operations.

---

### 10. 🟡 What are Global Query Filters in EF Core and what are common use cases?

Global Query Filters are LINQ predicates applied to all queries for an entity via `OnModelCreating`. Common uses: soft-delete (`IsDeleted == false`), multi-tenancy (`TenantId == currentTenantId`).

**Hint:** A great candidate knows that filters can be bypassed with `.IgnoreQueryFilters()` and that they also apply to navigation property loads (Include).

**🚩 Red Signal:** Manually adds `.Where(x => !x.IsDeleted)` to every query instead of using global filters.
