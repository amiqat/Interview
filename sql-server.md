# SQL Server — Senior Interview Questions

> **Time:** ~15 min · **Pick:** 4-6 questions · **Start with** 🟢 then go deeper

---

## Complex Queries

### 🟢 1. Explain CTEs (Common Table Expressions) and recursive CTEs.

**Hint:** CTE is a named temporary result set defined with `WITH`. Improves readability for complex queries. Scope: single statement only. Recursive CTE references itself — useful for hierarchies (org charts, tree structures, BOM). Must have an anchor member + recursive member with termination condition.

```sql
-- Recursive CTE: Employee hierarchy
WITH OrgChart AS (
    -- Anchor: top-level managers
    SELECT Id, Name, ManagerId, 0 AS Level
    FROM Employees WHERE ManagerId IS NULL
    UNION ALL
    -- Recursive: direct reports
    SELECT e.Id, e.Name, e.ManagerId, oc.Level + 1
    FROM Employees e
    INNER JOIN OrgChart oc ON e.ManagerId = oc.Id
)
SELECT * FROM OrgChart
OPTION (MAXRECURSION 100); -- Safety limit
```

**Follow-up:** What's the default max recursion? (100) What happens without `MAXRECURSION`? How does CTE performance compare to temp tables for large datasets?

**🚩 Red Signal:** Cannot write a basic CTE or confuses it with temp tables. Doesn't mention termination condition for recursive CTEs.

---

### 🟡 2. What are window functions and when do you use them?

**Hint:** `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()`, `LEAD()`, `LAG()`, `SUM() OVER(...)`, `NTILE()`. Operate on a window of rows related to the current row **without collapsing** them (unlike `GROUP BY`).

```sql
-- Running total + previous month comparison
SELECT
    OrderDate,
    Amount,
    SUM(Amount) OVER (ORDER BY OrderDate) AS RunningTotal,
    LAG(Amount) OVER (ORDER BY OrderDate) AS PreviousAmount,
    Amount - LAG(Amount) OVER (ORDER BY OrderDate) AS Change
FROM Orders;

-- Pagination
SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (ORDER BY CreatedAt DESC) AS RowNum
    FROM Products
) t WHERE RowNum BETWEEN 21 AND 40;
```

**Follow-up:** What's the difference between `RANK()`, `DENSE_RANK()`, and `ROW_NUMBER()`? When do you use `PARTITION BY` inside `OVER()`?

**🚩 Red Signal:** Uses self-joins or correlated subqueries for problems easily solved by window functions.

---

### 🟡 3. How do you de-duplicate rows efficiently?

**Hint:** Use `ROW_NUMBER()` in a CTE, then delete rows where `RowNum > 1`:

```sql
WITH Dupes AS (
    SELECT *, ROW_NUMBER() OVER (
        PARTITION BY Email ORDER BY CreatedAt DESC -- Keep newest
    ) AS RowNum
    FROM Customers
)
DELETE FROM Dupes WHERE RowNum > 1;
```

**Follow-up:** How do you decide which duplicate to keep? What if the table has billions of rows?

**🚩 Red Signal:** Uses cursors to loop through duplicates or has no strategy for choosing which row to keep.

---

### 🟡 4. Explain CROSS APPLY and OUTER APPLY.

**Hint:** `CROSS APPLY` is like `INNER JOIN` with a table-valued function or correlated subquery — returns rows only when the right side produces results. `OUTER APPLY` is like `LEFT JOIN` — returns NULLs when the right side has no rows. Useful for: top-N per group, unpivoting, calling TVFs.

```sql
-- Top 3 orders per customer
SELECT c.Name, o.OrderDate, o.Total
FROM Customers c
CROSS APPLY (
    SELECT TOP 3 OrderDate, Total
    FROM Orders WHERE CustomerId = c.Id
    ORDER BY Total DESC
) o;
```

**Follow-up:** Why can't you use a regular `JOIN` for this pattern? (The correlated subquery references the outer table.)

**🚩 Red Signal:** Has never used `APPLY` or always uses correlated subqueries without knowing this alternative.

---

## Indexes

### 🟢 5. What is the difference between clustered and non-clustered indexes?

**Hint:** Clustered index defines the physical order of data in the table (B-tree, leaf nodes = data pages). One per table, usually the PK. Non-clustered index is a separate B-tree structure with pointers (RID or clustered key) back to the data. Multiple allowed per table.

**Covering index:** Include all columns needed by a query using `INCLUDE` clause — avoids key lookups.

```sql
-- Covering index for a common query
CREATE NONCLUSTERED INDEX IX_Orders_CustomerId
ON Orders (CustomerId)
INCLUDE (OrderDate, Total, Status);
```

**Follow-up:** What is a key lookup? When does SQL Server choose an index scan vs index seek?

**🚩 Red Signal:** Thinks a table can have multiple clustered indexes or doesn't understand what a covering index is.

---

### 🟡 6. When would you use a filtered index?

**Hint:** Index with a `WHERE` clause — indexes only a subset of rows. Great for columns with many NULLs, skewed data distribution, or common filter predicates.

```sql
-- Only index active orders (90% of queries filter on this)
CREATE NONCLUSTERED INDEX IX_Orders_Active
ON Orders (CustomerId, OrderDate)
WHERE Status = 'Active';

-- Unique constraint with NULLs allowed
CREATE UNIQUE NONCLUSTERED INDEX IX_Users_Email
ON Users (Email)
WHERE Email IS NOT NULL;
```

**Follow-up:** What are the limitations of filtered indexes? (Can't be used as a foreign key, parameterized queries may not match, limited predicate syntax.)

**🚩 Red Signal:** Creates full indexes on columns where only a small subset of values is queried.

---

### 🟡 7. How do you identify and fix missing or unused indexes?

**Hint:**

| Need | Tool / DMV |
|------|-----------|
| Missing indexes | `sys.dm_db_missing_index_details`, `sys.dm_db_missing_index_group_stats`, execution plan warnings |
| Unused indexes | `sys.dm_db_index_usage_stats` — `user_seeks = 0 AND user_scans = 0` |
| Index health | `sys.dm_db_index_physical_stats` — fragmentation levels |
| Query analysis | Query Store, Extended Events, execution plans |

**Caution:** Missing index DMVs suggest single-column indexes without considering existing indexes. Always evaluate holistically.

**Follow-up:** What's the cost of too many indexes? (Write overhead on INSERT/UPDATE/DELETE, storage, memory for index pages.)

**🚩 Red Signal:** Adds indexes based on guesswork without checking execution plans or DMVs.

---

### 🟡 8. What is index fragmentation and how do you address it?

**Hint:** Logical fragmentation: out-of-order pages in the B-tree. Physical fragmentation: non-contiguous pages on disk.

| Fragmentation | Action |
|---------------|--------|
| < 10% | Do nothing |
| 10-30% | `ALTER INDEX REORGANIZE` (online, lightweight) |
| > 30% | `ALTER INDEX REBUILD` (can be online with Enterprise edition) |

Check with `sys.dm_db_index_physical_stats`. Consider **fill factor** for insert-heavy tables (leave space for new rows). Columnstore indexes have different fragmentation patterns.

**Follow-up:** What is the difference between online and offline index rebuild? When does `REORGANIZE` not help?

**🚩 Red Signal:** Rebuilds all indexes nightly without checking fragmentation levels.

---

### 🔴 9. Explain execution plans — what do you look for?

**Hint:** Key things to examine:
1. **Scans vs Seeks** — Scans read all rows; seeks use index efficiently
2. **Key Lookups** — Non-clustered index found the row but needs additional columns from clustered index → add `INCLUDE` columns
3. **Sort operators** — Expensive in-memory sorts indicate missing index with correct sort order
4. **Estimated vs Actual rows** — Large discrepancy indicates stale statistics or parameter sniffing
5. **Hash joins** — May indicate missing indexes on join columns
6. **Parallelism** — Not always good (overhead); `MAXDOP` hint if needed
7. **Thick arrows** — Large data flow between operators
8. **Warnings** — Missing indexes, implicit conversions, sorts spilling to tempdb

**Follow-up:** How do you force SQL Server to use a specific execution plan? (Plan guides, Query Store forced plans, `OPTION (USE PLAN)` hint.)

**🚩 Red Signal:** Never reads execution plans or only looks at "green checkmark" estimated plans.

---

## Large Data & Bulk Operations

### 🟢 10. How does `SqlBulkCopy` work and when should you use it?

**Hint:** Streams data directly into the table using TDS bulk insert protocol. Minimal logging with simple/bulk-logged recovery model. Much faster than row-by-row INSERT.

```csharp
using var bulkCopy = new SqlBulkCopy(connection)
{
    DestinationTableName = "Products",
    BatchSize = 5000,
    BulkCopyTimeout = 600
};

// Map columns
bulkCopy.ColumnMappings.Add("Name", "Name");
bulkCopy.ColumnMappings.Add("Price", "Price");

// Stream from DataReader, DataTable, or DbDataReader
await bulkCopy.WriteToServerAsync(dataTable);
```

**Key settings:** `BatchSize` (rows per batch — controls memory + lock duration), `BulkCopyTimeout`, `EnableStreaming` for large datasets, `SqlBulkCopyOptions.TableLock` for fastest insert.

**Follow-up:** How do you handle errors during bulk copy? What about identity columns? How does batch size affect lock escalation?

**🚩 Red Signal:** Inserts millions of rows with individual INSERT statements or EF `Add`/`SaveChanges` in a loop.

---

### 🟡 11. How do you perform an UPSERT in SQL Server?

**Hint:** Two approaches:

```sql
-- Approach 1: MERGE (use with caution)
MERGE INTO Products AS target
USING @StagingTable AS source ON target.SKU = source.SKU
WHEN MATCHED THEN
    UPDATE SET target.Price = source.Price, target.Name = source.Name
WHEN NOT MATCHED THEN
    INSERT (SKU, Name, Price) VALUES (source.SKU, source.Name, source.Price);

-- Approach 2: UPDATE + INSERT (safer, no MERGE bugs)
UPDATE p SET p.Price = s.Price, p.Name = s.Name
FROM Products p INNER JOIN @StagingTable s ON p.SKU = s.SKU;

INSERT INTO Products (SKU, Name, Price)
SELECT s.SKU, s.Name, s.Price FROM @StagingTable s
WHERE NOT EXISTS (SELECT 1 FROM Products p WHERE p.SKU = s.SKU);
```

**⚠️ `MERGE` pitfalls:** Known concurrency bugs (race conditions without proper locking). Always use `WITH (HOLDLOCK)` on the target. Microsoft has documented multiple bugs over the years.

**Follow-up:** How do you make the UPSERT safe under concurrency? What about OUTPUT clause for auditing?

**🚩 Red Signal:** Uses `MERGE` without locking hints or is unaware of its concurrency issues.

---

### 🟢 12. What are temp tables vs table variables and when do you use each?

**Hint:**

| Feature | Temp Table (`#temp`) | Table Variable (`@table`) |
|---------|---------------------|--------------------------|
| Storage | `tempdb` | `tempdb` (NOT memory!) |
| Statistics | ✅ Yes | ❌ No (before SQL 2019) |
| Indexes | ✅ Yes | Limited (PK/unique only) |
| Parallelism | ✅ Yes | ❌ No (pre-2019) |
| Recompilation | Triggers recompile on stats change | No recompile |
| Scope | Session / batch | Batch only |
| Estimated rows | Based on statistics | Always estimated as 1 row (pre-2019) |

**Rule of thumb:** Temp tables for large datasets (>100 rows), table variables for small datasets or when you want to avoid recompilation.

**SQL Server 2019+:** Table Variable Deferred Compilation improves table variable estimates.

**Follow-up:** What about global temp tables (`##temp`)? When are they useful? How do temp tables affect `tempdb` contention?

**🚩 Red Signal:** Always uses table variables for large datasets or thinks table variables live in memory.

---

### 🟡 13. What are Table-Valued Parameters (TVP)?

**Hint:** User-defined table types passed as parameters to stored procedures/functions. Replaces comma-delimited strings or XML for passing collections from application code. Strongly typed, multi-column. Readonly in the procedure.

```sql
-- Create type
CREATE TYPE dbo.ProductList AS TABLE (
    SKU NVARCHAR(50),
    Name NVARCHAR(200),
    Price DECIMAL(18,2)
);

-- Use in stored procedure
CREATE PROCEDURE dbo.UpsertProducts @Products dbo.ProductList READONLY AS
BEGIN
    MERGE INTO Products AS t
    USING @Products AS s ON t.SKU = s.SKU
    WHEN MATCHED THEN UPDATE SET t.Price = s.Price
    WHEN NOT MATCHED THEN INSERT (SKU, Name, Price) VALUES (s.SKU, s.Name, s.Price);
END;
```

```csharp
// C# usage
var table = new DataTable();
table.Columns.Add("SKU", typeof(string));
table.Columns.Add("Name", typeof(string));
table.Columns.Add("Price", typeof(decimal));

foreach (var p in products)
    table.Rows.Add(p.SKU, p.Name, p.Price);

var param = new SqlParameter("@Products", SqlDbType.Structured)
{
    TypeName = "dbo.ProductList",
    Value = table
};
await command.ExecuteNonQueryAsync();
```

**Follow-up:** TVP vs `SqlBulkCopy` — when to use which? (TVP for moderate datasets with logic; BulkCopy for raw speed with massive data.)

**🚩 Red Signal:** Passes large CSV strings and parses them inside stored procedures instead of using TVPs.

---

### 🟡 14. How do you handle large data reads without blocking or memory issues?

**Hint:**

| Strategy | When to use |
|----------|-------------|
| `SNAPSHOT` isolation | Consistent reads without blocking writers |
| `READ COMMITTED SNAPSHOT` | Database-level setting, versioned reads |
| `NOLOCK` / `READ UNCOMMITTED` | Accept dirty reads for reports |
| `DataReader` (streaming) | Avoid loading entire result into memory |
| `OFFSET/FETCH` pagination | Return data in pages |
| Read replicas / AG secondary | Offload reporting queries |

```csharp
// Stream large results — don't buffer
await using var reader = await command.ExecuteReaderAsync(
    CommandBehavior.SequentialAccess);
while (await reader.ReadAsync())
{
    yield return MapToEntity(reader);
}
```

**Follow-up:** What are the trade-offs of `SNAPSHOT` isolation? (Version store in `tempdb`, increased `tempdb` usage, potential for update conflicts.)

**🚩 Red Signal:** Reads millions of rows into a `DataTable` in memory, or uses `NOLOCK` everywhere without understanding dirty reads.

---

### 🔴 15. What is query plan caching and parameter sniffing?

**Hint:** SQL Server caches execution plans keyed by query text + parameters. **Parameter sniffing:** the plan is compiled/optimized for the first parameter values used. If data distribution is skewed, this plan may be terrible for other values.

**Example:** A query with `WHERE Status = @Status` — plan optimized for `'Active'` (1M rows, scan) is reused for `'Cancelled'` (10 rows, should seek).

**Mitigations:**
- `OPTION (RECOMPILE)` — fresh plan every time (compilation cost)
- `OPTION (OPTIMIZE FOR (@Status = 'Active'))` — hint for typical value
- `OPTION (OPTIMIZE FOR UNKNOWN)` — uses average statistics
- Query Store forced plans — pin a known good plan
- Plan guides — attach hints without modifying queries

**Follow-up:** How do you use Query Store to identify and fix parameter sniffing issues?

**🚩 Red Signal:** Doesn't know what parameter sniffing is, or always uses `RECOMPILE` without understanding the trade-off.

---

## Concurrency & Locking

### 🟡 16. What are the main types of locks in SQL Server?

**Hint:** Shared (S) — reads; Exclusive (X) — writes; Update (U) — read with intent to update (prevents deadlock). Lock granularity: row → page → table. Lock escalation: SQL Server may escalate to table lock when many row locks are held (threshold ~5000 locks).

**Follow-up:** How do you prevent lock escalation? (`ALTER TABLE ... SET (LOCK_ESCALATION = DISABLE)` — use carefully.)

**🚩 Red Signal:** Cannot explain the difference between shared and exclusive locks or how deadlocks occur.

---

### 🔴 17. How do you diagnose and prevent deadlocks?

**Hint:**
1. Enable deadlock graph via Extended Events or trace flag 1222
2. Common cause: two sessions lock resources in different order
3. Prevention: consistent lock ordering, keep transactions short, use proper indexes (fewer locks), consider `READ COMMITTED SNAPSHOT` to eliminate reader-writer blocking
4. Use `SET DEADLOCK_PRIORITY LOW` on less important sessions

**Follow-up:** Walk me through a real deadlock scenario you've resolved.

**🚩 Red Signal:** Thinks retry logic alone is sufficient without investigating root cause.

---

## Scenario Questions

### 💡 18. You need to load 10 million rows from a CSV file into SQL Server. Walk me through your approach.

**Expected approach:**
1. Stage to a temp/staging table (no constraints, no indexes)
2. Use `SqlBulkCopy` with `BatchSize` of 5,000-10,000 and `TableLock` option
3. Validate data in staging table
4. Use `MERGE` or `INSERT`/`UPDATE` from staging to target with proper locking
5. Consider switching to bulk-logged recovery model during load
6. Rebuild indexes after load
7. For ongoing loads: consider SSIS, ADF, or a streaming approach

**🚩 Red Signal:** Uses EF Core to insert row by row, or doesn't mention a staging table strategy.

---

### 💡 19. A query that normally takes 100ms suddenly takes 30 seconds. Nothing else changed. What happened?

**Expected approach:**
1. **Parameter sniffing** — bad cached plan. Check with `sys.dm_exec_query_stats` and compare estimated vs actual rows
2. **Statistics are stale** — `UPDATE STATISTICS` on relevant tables
3. **Lock/block** — check `sys.dm_exec_requests` for blocking session
4. **Index fragmentation** — check `sys.dm_db_index_physical_stats`
5. **tempdb contention** — heavy temp table usage or version store
6. **Resource pressure** — CPU, memory, disk I/O from other queries

**🚩 Red Signal:** Immediately suggests adding more hardware or restarting SQL Server.

---

### 💡 20. Design a schema for a multi-tenant SaaS application. How do you handle tenant data isolation?

**Expected approach:**
1. **Shared database, shared schema** — `TenantId` column on every table + row-level security
2. **Shared database, separate schema** — each tenant gets their own schema (e.g., `tenant1.Orders`)
3. **Separate database per tenant** — maximum isolation, complex management
4. Consider: row-level security policies, connection string routing, EF Core global query filters
5. Index considerations: `TenantId` should be leading column in most indexes

**Follow-up:** How do you handle cross-tenant reporting? Tenant-specific schema customizations?

**🚩 Red Signal:** No mention of security/isolation, or proposes separate databases for 10,000+ tenants.
