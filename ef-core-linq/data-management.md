# Data Management

---

### 1. 🟡 Your team needs to import 50,000 products from a supplier CSV file nightly. The current implementation adds each product to the `DbContext` and calls `SaveChanges()` — it takes over 10 minutes. How do you bring this down to seconds?

The default `SaveChanges()` sends one `INSERT` statement per entity, and the change tracker runs `DetectChanges()` on every `Add()` call — this is extremely slow at scale. Several strategies can reduce the time dramatically:
- **`SqlBulkCopy`** via the underlying `SqlConnection` is the fastest option — it streams rows directly using the TDS protocol's bulk insert path and can insert 50K rows in seconds.
- **Third-party libraries** like `EFCore.BulkExtensions` wrap `SqlBulkCopy` with EF-friendly APIs (`BulkInsertAsync`).
- **Batching** — if you must use `SaveChanges()`, disable `AutoDetectChangesEnabled`, use `AddRange()` instead of `Add()`, and call `SaveChanges()` every 500–1,000 entities, clearing the tracker between batches.
- **EF Core 7+ `ExecuteUpdate`/`ExecuteDelete`** help for set-based updates but don't solve bulk inserts directly.

```csharp
// ❌ One SaveChanges per entity — painfully slow
foreach (var product in products)
{
    context.Products.Add(product);
    await context.SaveChangesAsync();
}

// ✅ Batched with tracker management
context.ChangeTracker.AutoDetectChangesEnabled = false;
for (int i = 0; i < products.Count; i += 1000)
{
    context.Products.AddRange(products.Skip(i).Take(1000));
    await context.SaveChangesAsync();
    context.ChangeTracker.Clear();
}
```

**Hint:** Look for awareness that `Add` vs `AddRange` affects change-tracker performance (each `Add` triggers `DetectChanges`) and that clearing the tracker between batches prevents memory growth. Follow up: "How would you handle upserts (insert-or-update) in bulk?"

**🚩 Red Signal:** Suggests calling `SaveChanges()` inside a tight loop for each entity, or is completely unaware of `SqlBulkCopy` as an option.

---

### 2. 🟡 You deploy your EF Core application to production behind a load balancer with 3 API instances. Each instance calls `context.Database.MigrateAsync()` at startup. Occasionally deployments fail with race conditions when multiple instances start simultaneously. What is the safe production strategy for applying migrations?

`MigrateAsync()` at startup is convenient for single-instance development but causes problems in scaled-out deployments — multiple instances may try to apply the same migration concurrently, leading to deadlocks or duplicate-key errors in the `__EFMigrationsHistory` table. The safe production strategy is to **separate migration execution from application startup**:
- **Generate idempotent SQL scripts** with `dotnet ef migrations script --idempotent` and apply them through your CI/CD pipeline before deploying the new application code. Idempotent scripts check the `__EFMigrationsHistory` table before applying each migration, making them safe to re-run.
- **Migration bundles** (EF Core 6+) produce a self-contained executable that applies pending migrations. They are ideal for CI/CD environments where the .NET SDK is not available on the deployment agent.
- Run migrations as a **dedicated step** in your deployment pipeline (e.g., a Kubernetes init container or a CI/CD job) rather than letting each application instance race to migrate.

**Hint:** Look for understanding of why `MigrateAsync()` at startup is problematic at scale and awareness of idempotent scripts or migration bundles. Follow up: "How do you handle rollbacks if a migration breaks production?" and "What does the `__EFMigrationsHistory` table contain?"

**🚩 Red Signal:** Applies migrations manually in production without a scripted, repeatable process, or sees no issue with `MigrateAsync()` running on every instance at startup.

---

### 3. 🟡 Your EF Core query for a complex financial report with multiple joins, window functions, and aggregations is too slow — the LINQ translation produces suboptimal SQL. You need raw SQL performance but don't want to abandon the ORM entirely. How do you combine both approaches?

EF Core provides several ways to use raw SQL while keeping ORM benefits like mapping, tracking, and composability:
- **`FromSqlRaw` / `FromSqlInterpolated`** execute raw SQL and map results to entity types. Crucially, you can compose further LINQ on top (`.Where()`, `.OrderBy()`, `.Include()`), so you get raw SQL for the heavy lifting and LINQ for the last-mile filtering.
- **`SqlQuery<T>`** (EF Core 7+) maps raw SQL results to arbitrary types (not just entities), which is perfect for report DTOs.
- **`ExecuteSqlRaw` / `ExecuteSqlInterpolated`** handle non-query commands (INSERT, UPDATE, DELETE).

```csharp
// Raw SQL composed with LINQ — best of both worlds
var report = await context.Orders
    .FromSqlInterpolated(
        $"SELECT * FROM Orders WHERE Total > {threshold}")
    .Where(o => o.Status == OrderStatus.Completed)
    .OrderBy(o => o.CreatedDate)
    .ToListAsync();
```

Always use parameterised queries — `FromSqlInterpolated` uses `FormattableString` for safe parameter binding. Never concatenate user input into SQL strings. For views or stored procedures that don't map to entities, `SqlQuery<T>` with a DTO gives you type safety without entity configuration overhead.

**Hint:** Look for emphasis on SQL injection prevention via parameterised queries. A strong candidate explains the composability of `FromSqlRaw` (you can chain LINQ after it) and when to fall back to `SqlQuery<T>` for non-entity projections. Follow up: "What are the limitations of `FromSqlRaw` composability — when does it not work?"

**🚩 Red Signal:** Concatenates user input directly into raw SQL strings, or is unaware that raw SQL results can be composed with further LINQ.

---

### 4. 🟡 Two customer-support agents open the same customer record simultaneously. Agent A changes the email, Agent B changes the phone number. Agent B saves last and silently overwrites Agent A's email change. How do you detect and handle this with EF Core?

This is a lost-update problem that EF Core solves with **optimistic concurrency**. You add a concurrency token — typically a `[Timestamp]` property that maps to a SQL Server `rowversion` column (or a `[ConcurrencyCheck]` on a specific column). EF Core includes the token's original value in the `WHERE` clause of the `UPDATE` statement. If another user modified the row since it was read, the `WHERE` matches zero rows and EF Core throws `DbUpdateConcurrencyException`. You then handle the conflict by reloading the entity, merging changes, and retrying.

```csharp
// Entity with concurrency token
public class Customer
{
    public int Id { get; set; }
    public string Email { get; set; }
    public string Phone { get; set; }

    [Timestamp]
    public byte[] RowVersion { get; set; }
}

// Handling the conflict
try
{
    await context.SaveChangesAsync();
}
catch (DbUpdateConcurrencyException ex)
{
    var entry = ex.Entries.Single();
    var dbValues = await entry.GetDatabaseValuesAsync();
    // Compare entry.OriginalValues, entry.CurrentValues, and dbValues
    // Merge or present conflict to user, then retry
    entry.OriginalValues.SetValues(dbValues);
    await context.SaveChangesAsync();
}
```

The resolution strategy depends on business requirements: you might auto-merge non-conflicting fields, present a diff to the user, or apply a last-writer-wins policy per field rather than per row.

**Hint:** Look for understanding of optimistic vs pessimistic concurrency and the specific EF Core mechanism (`RowVersion`/`ConcurrencyCheck`, the `WHERE` clause check, `DbUpdateConcurrencyException`). Follow up: "When would you choose pessimistic locking instead?" and "How does `entry.GetDatabaseValues()` help with merge resolution?"

**🚩 Red Signal:** Has no strategy for handling concurrent updates, or relies solely on database-level locking (pessimistic concurrency) for all operations without considering the scalability impact.

---

### 5. 🟡 Your multi-tenant SaaS application stores all tenants in a single database with a `TenantId` column on every table. Developers keep forgetting to add `.Where(x => x.TenantId == currentTenantId)` to their queries, and you've had two data-leak incidents where Tenant A saw Tenant B's data. How do you enforce tenant isolation automatically so this can never happen again?

Use **Global Query Filters** in EF Core. In `OnModelCreating`, you configure a LINQ predicate on each entity that EF Core automatically appends to every query for that entity type — including queries through navigation properties (`.Include()`). This eliminates the possibility of developers forgetting the filter.

```csharp
public class AppDbContext : DbContext
{
    private readonly int _tenantId;

    public AppDbContext(DbContextOptions options, ITenantProvider tenant)
        : base(options) => _tenantId = tenant.CurrentTenantId;

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Product>()
            .HasQueryFilter(p => p.TenantId == _tenantId);

        modelBuilder.Entity<Order>()
            .HasQueryFilter(o => o.TenantId == _tenantId);
    }
}
```

The filter applies transparently to all LINQ queries, `Include()` calls, and even `FromSqlRaw` when composed with LINQ. For admin scenarios where you need cross-tenant access (e.g., a global report), you can bypass filters with `.IgnoreQueryFilters()`. It's important to restrict usage of `IgnoreQueryFilters()` via code reviews or a custom Roslyn analyzer to prevent accidental bypasses. Note that each entity can only have one global filter, so if you need both soft-delete and multi-tenancy, combine them into a single predicate expression.

**Hint:** A strong candidate knows that filters apply to `Include()` (so related entities are also filtered), that `.IgnoreQueryFilters()` is the escape hatch, and that only one filter per entity is allowed (so combine predicates if needed). Follow up: "How would you test that the filter is applied correctly?" and "What happens if you forget to configure the filter on a new entity?"

**🚩 Red Signal:** Manually adds `.Where(x => x.TenantId == currentTenantId)` to every query instead of using global filters, or is completely unaware of the feature.
