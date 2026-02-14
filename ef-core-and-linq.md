# EF Core & LINQ — Senior Interview Questions

> **Time:** ~15 min · **Pick:** 4-6 questions · **Start with** 🟢 then go deeper

---

## EF Core Fundamentals

### 🟢 1. What is the difference between tracking and no-tracking queries?

**Hint:** Tracking queries attach entities to the `ChangeTracker` for update detection. No-tracking (`.AsNoTracking()`) skips identity resolution and change detection — faster for read-only scenarios. `.AsNoTrackingWithIdentityResolution()` is a middle ground.

```csharp
// Read-only — fast
var users = await context.Users.AsNoTracking().ToListAsync();

// Need to update — tracked
var user = await context.Users.FirstAsync(u => u.Id == id);
user.Name = "Updated";
await context.SaveChangesAsync();
```

**Follow-up:** What is `.AsNoTrackingWithIdentityResolution()` and when would you use it? (When you need no-tracking but want to avoid duplicate instances of the same entity in the result.)

**🚩 Red Signal:** Always uses tracking queries even for read-only endpoints, or doesn't know `.AsNoTracking()` exists.

---

### 🟢 2. How does the Change Tracker work?

**Hint:** EF snapshots entity property values on load (or attach). On `SaveChanges()`, it compares current values vs snapshot to detect changes. Entity states: `Added`, `Modified`, `Deleted`, `Unchanged`, `Detached`.

**Follow-up:** How do you manually set entity state? (`context.Entry(entity).State = EntityState.Modified`) When would you need to? (Disconnected scenarios, e.g., web APIs where the entity was deserialized from a request.)

**🚩 Red Signal:** Cannot explain entity states or thinks `SaveChanges()` saves everything regardless of changes.

---

### 🟡 3. What are owned types and when would you use them?

**Hint:** Value objects mapped to the same table (or a separate table) as the owner. Configured via `OwnsOne` / `OwnsMany`. Useful for DDD value objects like `Address`, `Money`, `DateRange`. No separate identity — always accessed through the owner.

```csharp
modelBuilder.Entity<Order>().OwnsOne(o => o.ShippingAddress, sa =>
{
    sa.Property(a => a.Street).HasColumnName("ShippingStreet");
    sa.Property(a => a.City).HasColumnName("ShippingCity");
});
```

**Follow-up:** How does owned type mapping differ for `OwnsOne` vs `OwnsMany`? (One maps to same table by default; Many maps to a separate table.)

**🚩 Red Signal:** Creates a full entity with its own key for simple value objects instead of using owned types.

---

### 🟡 4. How do you handle concurrency conflicts in EF Core?

**Hint:** Use a `[ConcurrencyCheck]` column or a `[Timestamp]`/`rowversion` column. EF includes the original value in the `WHERE` clause on update. If zero rows affected → `DbUpdateConcurrencyException`.

```csharp
// Entity
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; }

    [Timestamp]
    public byte[] RowVersion { get; set; }
}

// Handling conflict
try { await context.SaveChangesAsync(); }
catch (DbUpdateConcurrencyException ex)
{
    var entry = ex.Entries.Single();
    var dbValues = await entry.GetDatabaseValuesAsync();
    // Resolve: client wins, store wins, or merge
}
```

**Follow-up:** What are the three resolution strategies? When would you pick each?

**🚩 Red Signal:** No strategy for concurrency — just "last write wins" without acknowledging data loss risk.

---

## Performance

### 🟢 5. Explain the N+1 query problem and how to avoid it.

**Hint:** Loading a collection then lazily loading related entities produces 1 + N SQL queries. Example: load 100 orders, then for each order load its items = 101 queries.

**Solutions:**
- `.Include()` / `.ThenInclude()` — eager loading (single or split query)
- `.AsSplitQuery()` — avoids cartesian explosion with multiple includes
- Projection (`.Select()`) — load only needed fields
- Explicit loading (`context.Entry(order).Collection(o => o.Items).Load()`)

**Follow-up:** What is cartesian explosion and how does `.AsSplitQuery()` solve it? What's the tradeoff? (Split query runs multiple SQL statements — not atomic, potential inconsistency.)

**🚩 Red Signal:** Relies on lazy loading in production APIs without realizing the performance impact.

---

### 🟡 6. What are compiled queries and when are they useful?

**Hint:** `EF.CompileQuery()` / `EF.CompileAsyncQuery()` pre-compiles the LINQ expression tree into a cached delegate. Eliminates expression tree compilation cost on every execution.

```csharp
private static readonly Func<AppDbContext, int, Task<User?>> GetUserById =
    EF.CompileAsyncQuery((AppDbContext ctx, int id) =>
        ctx.Users.FirstOrDefault(u => u.Id == id));

// Usage
var user = await GetUserById(context, userId);
```

**Follow-up:** In what scenario is the compilation cost actually noticeable? (High-throughput hot paths executing the same parameterized query thousands of times per second.)

**🚩 Red Signal:** Compiles every query, or doesn't know the feature exists when performance is critical.

---

### 🟡 7. How do you use raw SQL and when is it appropriate?

**Hint:** `FromSqlRaw()` / `FromSqlInterpolated()` for queries that map to entities. `ExecuteSqlRaw()` / `ExecuteSqlInterpolated()` for non-query commands. Use when LINQ can't express the query efficiently (complex CTEs, window functions, database-specific features).

```csharp
// Safe — parameterized
var users = await context.Users
    .FromSqlInterpolated($"SELECT * FROM Users WHERE Name = {name}")
    .ToListAsync();

// DANGEROUS — SQL injection!
var users = await context.Users
    .FromSqlRaw($"SELECT * FROM Users WHERE Name = '{name}'")
    .ToListAsync();
```

**Follow-up:** What's the difference between `FromSqlRaw` and `FromSqlInterpolated`? Why is the raw version dangerous with string interpolation?

**🚩 Red Signal:** Concatenates user input into raw SQL strings, or never uses raw SQL even when LINQ generates terrible queries.

---

### 🔴 8. What are EF Core interceptors and how would you use them?

**Hint:** Interceptors hook into EF operations at low level: `DbCommandInterceptor` (before/after SQL execution), `SaveChangesInterceptor` (before/after SaveChanges), `DbConnectionInterceptor`. Use for: audit logging, soft deletes, query hints, multi-tenancy filters, read/write splitting.

```csharp
public class SlowQueryInterceptor : DbCommandInterceptor
{
    public override ValueTask<DbDataReader> ReaderExecutedAsync(
        DbCommand command, CommandExecutedEventData eventData,
        DbDataReader result, CancellationToken ct = default)
    {
        if (eventData.Duration.TotalMilliseconds > 500)
            Log.Warning("Slow query ({Duration}ms): {Sql}",
                eventData.Duration.TotalMilliseconds, command.CommandText);
        return new ValueTask<DbDataReader>(result);
    }
}
```

**Follow-up:** How do interceptors differ from global query filters? (Filters modify the LINQ query; interceptors hook into the ADO.NET pipeline.)

**🚩 Red Signal:** Not aware of interceptors, or implements cross-cutting concerns by modifying every repository method individually.

---

## Migrations

### 🟡 9. How do migrations work and what pitfalls exist in team environments?

**Hint:** Migrations are incremental schema diffs stored as C# code. Generated with `dotnet ef migrations add`. Applied via `dotnet ef database update` or `context.Database.Migrate()`.

**Pitfalls:**
- Merge conflicts in the model snapshot file
- Ordering issues when multiple developers add migrations concurrently
- Data loss from destructive migrations (drop column, change type)
- Long-running migrations blocking production deployments

**Best practices:** Always review generated SQL (`dotnet ef migrations script`), test on staging, use idempotent scripts, consider separate data migrations.

**Follow-up:** How do you handle migrations in CI/CD? How do you roll back a bad migration?

**🚩 Red Signal:** Blindly runs migrations in production without reviewing the generated SQL or testing on staging.

---

## LINQ

### 🟢 10. What is deferred execution in LINQ?

**Hint:** The query is NOT executed until the result is enumerated (`foreach`, `.ToList()`, `.FirstOrDefault()`). Allows query composition — chain `.Where()`, `.OrderBy()`, `.Select()` — only one SQL statement generated.

**Follow-up:** What are the dangers of deferred execution? (Multiple enumeration of the same `IQueryable`, modifying the source during iteration, disposed DbContext.)

**🚩 Red Signal:** Doesn't understand that calling `.Where()` does not hit the database, or enumerates an `IQueryable` multiple times.

---

### 🟢 11. What is the difference between `IEnumerable<T>` and `IQueryable<T>`?

**Hint:** `IEnumerable<T>` evaluates in memory (LINQ to Objects, uses delegates). `IQueryable<T>` translates expression trees to SQL (LINQ to provider). Casting `IQueryable` to `IEnumerable` causes client-side evaluation — **full table loaded into memory**.

```csharp
// BAD — loads ALL users then filters in memory
IEnumerable<User> GetUsers(AppDbContext ctx) => ctx.Users;
var active = GetUsers(ctx).Where(u => u.IsActive); // Client evaluation!

// GOOD — filter applied in SQL
IQueryable<User> GetUsers(AppDbContext ctx) => ctx.Users;
var active = GetUsers(ctx).Where(u => u.IsActive); // SQL WHERE clause
```

**Follow-up:** What happens to client evaluation warnings in EF Core 3.0+? (Throws exception by default instead of silent client evaluation.)

**🚩 Red Signal:** Returns `IEnumerable<T>` from repositories then applies `.Where()` — causes client evaluation on full dataset.

---

### 🟡 12. How does `Select` (projection) help performance in EF Core?

**Hint:** Projecting to a DTO with `.Select()` generates SQL that fetches only the needed columns. Avoids tracking, avoids loading full entity graphs. Often better than `.Include()` for read-only.

```csharp
// BAD — loads full entities + all navigation properties
var orders = await context.Orders
    .Include(o => o.Items).Include(o => o.Customer)
    .ToListAsync();

// GOOD — loads only what's needed
var orderDtos = await context.Orders
    .Select(o => new OrderDto
    {
        Id = o.Id,
        CustomerName = o.Customer.Name,
        TotalItems = o.Items.Count,
        Total = o.Items.Sum(i => i.Price)
    })
    .ToListAsync();
```

**Follow-up:** Does projection bypass global query filters? (No — filters still apply. You need `IgnoreQueryFilters()` to bypass.)

**🚩 Red Signal:** Always loads full entities and maps to DTOs in application code instead of projecting at the query level.

---

### 🟡 13. Explain `GroupBy` translation in EF Core.

**Hint:** EF Core translates `GroupBy` to SQL `GROUP BY` when possible. Since EF Core 3.0+, untranslatable LINQ throws `InvalidOperationException` instead of silently falling back to client evaluation. Aggregates (`.Sum()`, `.Count()`, `.Average()`) after `GroupBy` must be translatable.

**Follow-up:** Give an example of a `GroupBy` that can't be translated to SQL. (Grouping by a computed CLR property that has no SQL equivalent.)

**🚩 Red Signal:** Writes complex `GroupBy` operations without checking the generated SQL.

---

### 🟢 14. What is the difference between `First`, `FirstOrDefault`, `Single`, and `SingleOrDefault`?

**Hint:**

| Method | Empty | One | Many |
|--------|-------|-----|------|
| `First` | ❌ Throws | ✅ Returns | ✅ Returns first |
| `FirstOrDefault` | ✅ Default | ✅ Returns | ✅ Returns first |
| `Single` | ❌ Throws | ✅ Returns | ❌ Throws |
| `SingleOrDefault` | ✅ Default | ✅ Returns | ❌ Throws |

**Rule of thumb:** Use `Single` when uniqueness is a business invariant (e.g., lookup by unique email). Use `First` when you want the first match from an ordered set.

**Follow-up:** What SQL does `Single` vs `First` generate? (`TOP 2` vs `TOP 1` — `Single` needs to verify only one row exists.)

**🚩 Red Signal:** Uses `First` everywhere when `Single` is semantically correct, or doesn't know the difference.

---

### 🔴 15. What are expression trees and how does EF Core use them?

**Hint:** Expression trees (`Expression<Func<T, bool>>`) represent code as data — a tree of nodes that can be inspected and translated. EF Core's LINQ provider walks the expression tree and generates SQL. This is why `IQueryable` uses `Expression<>` not plain `Func<>`.

```csharp
// This can be translated to SQL
Expression<Func<User, bool>> filter = u => u.IsActive && u.Age > 18;

// This CANNOT — opaque delegate, forces client evaluation
Func<User, bool> filter = u => u.IsActive && u.Age > 18;
```

**Follow-up:** How would you build dynamic filters at runtime using expression trees? (Using `Expression.Lambda`, `Expression.AndAlso`, or libraries like `LINQKit`'s `PredicateBuilder`.)

**🚩 Red Signal:** Cannot explain why `Func<>` causes client evaluation but `Expression<>` doesn't.

---

## Scenario Questions

### 💡 16. Your API endpoint is slow. EF logs show 150 SQL queries for a single request. What do you do?

**Expected approach:**
1. Identify N+1 problem — likely lazy loading or missing `.Include()`
2. Check if `.Include()` causes cartesian explosion → try `.AsSplitQuery()`
3. Consider projection (`.Select()`) to load only needed fields
4. Enable EF Core logging (`optionsBuilder.LogTo(Console.WriteLine)`) or use MiniProfiler
5. Consider compiled queries for hot paths
6. Evaluate if a raw SQL query would be simpler and more efficient

**🚩 Red Signal:** Adds caching on top of the slow query without fixing the underlying problem.

---

### 💡 17. You need to insert 50,000 records efficiently using EF Core. How?

**Expected approach:**
1. **Don't** call `Add()` + `SaveChanges()` in a loop (one INSERT per row)
2. Use `AddRange()` + `SaveChanges()` in batches (e.g., 1,000 at a time) — EF Core batches INSERT statements
3. For maximum performance, bypass EF: use `SqlBulkCopy` directly
4. Consider third-party libraries: `EFCore.BulkExtensions` for `BulkInsert()`, `BulkUpdate()`, `BulkMerge()`
5. Disable `AutoDetectChangesEnabled` during bulk operations

```csharp
context.ChangeTracker.AutoDetectChangesEnabled = false;
foreach (var batch in records.Chunk(1000))
{
    context.Products.AddRange(batch);
    await context.SaveChangesAsync();
    context.ChangeTracker.Clear(); // Free tracked entities
}
```

**🚩 Red Signal:** Calls `SaveChanges()` inside a foreach loop for each individual entity.

---

### 💡 18. How do you implement soft deletes across all entities in EF Core?

**Expected approach:**
1. Add `IsDeleted` property (or `DeletedAt` timestamp) to entities via a base class or interface
2. Configure global query filter: `modelBuilder.Entity<Order>().HasQueryFilter(o => !o.IsDeleted)`
3. Override `SaveChanges()` to intercept `Deleted` state and change to `Modified` with `IsDeleted = true`
4. Or use a `SaveChangesInterceptor` for cleaner separation
5. Use `.IgnoreQueryFilters()` when you need to query deleted records (e.g., admin panel, audit)

**🚩 Red Signal:** Manually adds `WHERE IsDeleted = false` to every query instead of using global query filters.
