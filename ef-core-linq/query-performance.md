# Query Performance

---

### 1. 🟢 What is the N+1 query problem in one sentence?

Loading a list of parents and then accessing a navigation property on each one triggers N separate queries for the children — one per parent — instead of a single query that fetches everything.

**Hint:** The candidate should immediately connect this to lazy loading and mention `.Include()` as the fix.

**🚩 Red Signal:** Cannot define N+1 or confuses it with a different performance issue.

---

### 2. 🟢 What does `AsNoTracking()` do and when should you use it?

It disables EF Core's change tracker for the query results, eliminating the overhead of creating snapshots for each entity. Use it for read-only queries where you never intend to call `SaveChanges()` on the returned entities.

**Hint:** Look for awareness that the change tracker has a real CPU and memory cost, especially for large result sets.

**🚩 Red Signal:** Cannot explain what the change tracker does or when skipping it is appropriate.

---

### 3. 🟡 A LINQ query that worked perfectly in development suddenly throws an `InvalidOperationException` in production after upgrading from EF Core 2.x to 5+. The developer didn't change any application code. What changed in the framework and how do you fix the query?

In EF Core 2.x, expressions that couldn't be translated to SQL were silently evaluated on the client — the framework would pull all rows into memory and apply the filter in C#. This often caused severe performance problems that went unnoticed. Starting with EF Core 3.0+, untranslatable expressions throw an `InvalidOperationException` by default to make these issues visible. To fix the query, rewrite the untranslatable portion so EF Core can convert it to SQL, or explicitly move the client-side logic after a `.ToList()` / `.AsEnumerable()` call so the boundary is intentional. Use `.ToQueryString()` during development to verify that the entire query translates. If the expression involves a C# method with no SQL equivalent (e.g., a custom string helper), replace it with an EF-supported function or compute the value before the query.

**Hint:** Look for understanding of the `IQueryable` vs `IEnumerable` boundary and the danger of accidental client evaluation. A strong candidate mentions `ToQueryString()` for inspecting generated SQL and can explain why the breaking change was a net positive for production safety. Follow up: "How would you find all client-evaluated queries in an existing codebase after an upgrade?"

**🚩 Red Signal:** Believes all C# lambda expressions are automatically translatable to SQL, or is unaware that client evaluation was silently happening in EF Core 2.x.

---

### 4. 🟢 Your API endpoint returns 1,000 products in a read-only list, but response times are slower than expected. Profiling shows most of the time is spent inside the EF Core change tracker, not in the database. What's happening and how do you optimize this?

By default, EF Core tracks every entity returned by a query in its change tracker — it creates snapshots of each entity's property values so it can detect modifications at `SaveChanges()` time. For 1,000 products in a read-only endpoint, this snapshot creation and identity-resolution overhead is wasted work. Adding `.AsNoTracking()` to the query disables tracking, which eliminates the snapshot cost and reduces memory allocations significantly. For cases where you need identity resolution but not full change tracking, EF Core offers `.AsNoTrackingWithIdentityResolution()`. You can also set `QueryTrackingBehavior.NoTracking` at the `DbContext` level for contexts that are predominantly read-only.

```csharp
// ❌ Tracked by default — unnecessary overhead for read-only
var products = await context.Products.ToListAsync();

// ✅ No tracking — faster for read-only scenarios
var products = await context.Products.AsNoTracking().ToListAsync();
```

**Hint:** Look for awareness of what the change tracker actually stores (original-value snapshots, state entries) and why that has CPU/memory cost. Bonus: the candidate mentions `AsNoTrackingWithIdentityResolution()` or setting `NoTracking` as the default on the DbContext. Follow up: "When would `AsNoTracking` cause incorrect behaviour?"

**🚩 Red Signal:** Cannot explain what the change tracker does or why tracking has a memory/CPU cost for read-only queries.

---

### 5. 🟢 You're building an order history page that displays 100 orders with their line items. The page loads, but SQL Profiler shows 101 separate queries — one for orders and one for each order's items. What's happening and how do you fix it?

This is the classic N+1 query problem. The first query loads 100 orders, and then as each order's `Items` navigation property is accessed (via lazy loading), EF Core issues a separate SQL query for each order — producing 100 additional queries. The fix depends on the scenario:
- **Eager loading** with `.Include(o => o.Items)` joins items into the original query.
- **Split queries** via `.AsSplitQuery()` (EF Core 5+) issue two separate queries (one for orders, one for all items) instead of a single large JOIN, which avoids cartesian explosion when multiple collections are included.
- **Projection** with `.Select()` to fetch only the columns you need into a DTO, which also avoids loading full entity graphs.

```csharp
// ❌ N+1 — lazy loading fires per order
var orders = await context.Orders.ToListAsync();
foreach (var order in orders)
    Console.WriteLine(order.Items.Count); // triggers query each time

// ✅ Eager loading — single JOIN query
var orders = await context.Orders
    .Include(o => o.Items)
    .ToListAsync();
```

**Hint:** A great candidate discusses the trade-off between a single large JOIN (cartesian explosion when including multiple collections) vs `.AsSplitQuery()` (multiple round-trips but no duplication). Follow up: "How would you detect N+1 issues in a large codebase before they hit production?"

**🚩 Red Signal:** Relies entirely on lazy loading without understanding the performance implications, or doesn't know about `.Include()`.

---

### 6. 🔴 Your hot-path endpoint runs the same parameterised EF Core query roughly 10,000 times per second. A CPU profile shows 15% of time is spent compiling the LINQ expression tree — not executing the query against the database. How do you eliminate this overhead?

Use `EF.CompileQuery` or `EF.CompileAsyncQuery` to pre-compile the LINQ expression tree into a reusable delegate. This removes the cost of parsing and translating the expression tree on every invocation — the compiled delegate goes straight to SQL generation with cached metadata. EF Core 6+ already has an internal query cache that helps for most queries, but compiled queries bypass the cache lookup entirely, which matters when profiling proves LINQ compilation is the bottleneck. The compiled query is typically stored as a `static` field and reused across requests.

```csharp
// Compiled query — expression tree is processed once
private static readonly Func<AppDbContext, int, Task<Product?>> _getProduct =
    EF.CompileAsyncQuery((AppDbContext ctx, int id) =>
        ctx.Products.FirstOrDefault(p => p.Id == id));

// Usage — no expression tree overhead per call
var product = await _getProduct(context, productId);
```

The key distinction is between *query compilation time* (translating LINQ → SQL, which happens in the application) and *query execution time* (running the SQL on the database server). Compiled queries only help with the former. If profiling shows the bottleneck is on the database side, compiled queries won't help — you need query tuning or indexing instead.

**Hint:** Look for the candidate to clearly separate LINQ compilation overhead from database execution time. They should mention that EF Core's built-in query cache covers most scenarios and that compiled queries are an optimization for measured hot paths, not a default practice. Follow up: "What are the limitations of compiled queries — what LINQ features can't you use with them?"

**🚩 Red Signal:** Cannot explain the difference between query compilation time and database execution time, or suggests compiled queries as a blanket optimization for all queries.

---

### 7. 🟢 A developer writes this code to search products by category and is confused why it fetches all 2 million rows from the database before filtering. What's wrong?

```csharp
IEnumerable<Product> products = context.Products;
var filtered = products.Where(p => p.CategoryId == 5).ToList();
```

The problem is the variable type. When the developer declares `products` as `IEnumerable<Product>`, the subsequent `.Where()` call uses `System.Linq.Enumerable.Where` (in-memory LINQ) instead of `System.Linq.Queryable.Where` (expression-tree LINQ). This means EF Core materialises all 2 million products first and then filters in memory. The fix is to keep the type as `IQueryable<Product>` — or simply use `var` — so the `.Where()` builds an expression tree that EF Core translates into a SQL `WHERE` clause.

```csharp
// ❌ IEnumerable forces client-side evaluation
IEnumerable<Product> products = context.Products;
var filtered = products.Where(p => p.CategoryId == 5).ToList();

// ✅ IQueryable keeps the query server-side
IQueryable<Product> products = context.Products;
var filtered = products.Where(p => p.CategoryId == 5).ToList();
```

The same problem happens when calling `.ToList()` too early in a query chain — everything after the materialisation point runs in memory. As a rule, keep the query as `IQueryable` for as long as possible, apply all filters and projections, and materialise only at the end.

**Hint:** Look for understanding that `IQueryable<T>` builds an expression tree translated to SQL, while `IEnumerable<T>` runs LINQ-to-Objects in memory. Follow up: "How would you enforce this rule in code reviews or with static analysis?" and "What other common patterns accidentally trigger client evaluation?"

**🚩 Red Signal:** Uses `.ToList()` immediately after `DbSet` and then applies further LINQ filtering in memory on large datasets without realising the performance impact.
