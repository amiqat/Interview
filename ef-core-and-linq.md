# EF Core & LINQ — Senior Interview Questions

## EF Core

### 1. What is the difference between tracking and no-tracking queries?

**Hint:** Tracking queries attach entities to the `ChangeTracker` for update detection. No-tracking (`.AsNoTracking()`) is faster for read-only scenarios — no identity resolution, no change detection overhead.

**🚩 Red Signal:** Always uses tracking queries even for read-only endpoints, or doesn't know `.AsNoTracking()` exists.

---

### 2. How does the Change Tracker work?

**Hint:** EF snapshots entity state on load. On `SaveChanges()`, it compares current values vs snapshot to detect modifications. States: `Added`, `Modified`, `Deleted`, `Unchanged`, `Detached`.

**🚩 Red Signal:** Cannot explain entity states or thinks `SaveChanges()` saves everything regardless of changes.

---

### 3. What are owned types and when would you use them?

**Hint:** Value objects mapped to the same table (or a separate table) as the owner. Configured via `OwnsOne` / `OwnsMany`. Useful for DDD value objects like `Address`, `Money`. No separate identity.

**🚩 Red Signal:** Creates a full entity with its own key for simple value objects instead of using owned types.

---

### 4. How do you handle concurrency conflicts in EF Core?

**Hint:** Use a `[ConcurrencyCheck]` column or a `[Timestamp]`/`rowversion` column. EF includes the original value in the `WHERE` clause on update. Catch `DbUpdateConcurrencyException` and resolve (client wins, store wins, or merge).

**🚩 Red Signal:** No strategy for concurrency — just "last write wins" without acknowledging data loss risk.

---

### 5. Explain the N+1 query problem and how to avoid it.

**Hint:** Loading a collection and then lazily loading related entities produces 1 + N queries. Fix with `.Include()` (eager loading), explicit loading, or projection (`.Select()`). Split queries (`.AsSplitQuery()`) help with cartesian explosion.

**🚩 Red Signal:** Relies on lazy loading in production APIs without realizing the performance impact.

---

### 6. What are compiled queries and when are they useful?

**Hint:** `EF.CompileQuery()` / `EF.CompileAsyncQuery()` pre-compiles the LINQ expression tree into a cached delegate. Saves expression tree compilation cost on hot paths. Useful for frequently executed, parameterized queries.

**🚩 Red Signal:** Compiles every query or doesn't know the feature exists when performance is critical.

---

### 7. How do migrations work and what pitfalls exist in team environments?

**Hint:** Migrations are incremental schema diffs stored as C# code. Apply via `dotnet ef database update` or `context.Database.Migrate()`. Pitfalls: merge conflicts in snapshot, ordering issues, data loss from destructive migrations. Always review generated SQL.

**🚩 Red Signal:** Blindly runs migrations in production without reviewing the generated SQL or testing on staging.

---

## LINQ

### 8. What is deferred execution in LINQ?

**Hint:** The query is not executed until the result is enumerated (e.g., `foreach`, `.ToList()`, `.FirstOrDefault()`). Allows query composition. Beware of multiple enumeration of the same `IQueryable`.

**🚩 Red Signal:** Doesn't understand that calling `.Where()` does not hit the database, or enumerates an `IQueryable` multiple times.

---

### 9. What is the difference between `IEnumerable<T>` and `IQueryable<T>`?

**Hint:** `IEnumerable<T>` evaluates in memory (LINQ to Objects). `IQueryable<T>` translates expression trees to SQL (LINQ to SQL/EF). Casting `IQueryable` to `IEnumerable` causes client-side evaluation — full table loaded into memory.

**🚩 Red Signal:** Returns `IEnumerable<T>` from repositories and then applies `.Where()` filters — causes client evaluation on full dataset.

---

### 10. How does `Select` (projection) help performance in EF Core?

**Hint:** Projecting to a DTO with `.Select()` generates a SQL query that fetches only the needed columns. Avoids tracking, avoids loading entire entity graphs. Often better than `.Include()` for read-only scenarios.

**🚩 Red Signal:** Always loads full entities and maps to DTOs in memory instead of projecting at the query level.

---

### 11. Explain `GroupBy` behavior in EF Core.

**Hint:** EF Core translates `GroupBy` to SQL when possible. If it can't, it throws (EF Core 3.0+) or silently evaluates on client (older versions). Aggregates after `GroupBy` (e.g., `.Sum()`, `.Count()`) must be translatable.

**🚩 Red Signal:** Uses complex `GroupBy` operations that silently fall back to client evaluation and doesn't check generated SQL.

---

### 12. What is the difference between `First`, `FirstOrDefault`, `Single`, and `SingleOrDefault`?

**Hint:** `First` — first element, throws if empty. `FirstOrDefault` — first or default, no exception. `Single` — exactly one element, throws if zero or more than one. `SingleOrDefault` — zero or one, throws if more than one. Use `Single` when uniqueness is a business rule.

**🚩 Red Signal:** Uses `First` everywhere when `Single` is semantically correct, or doesn't know the difference.
