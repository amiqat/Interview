# Bulk Operations

---

### 1. 🟡 You need to load 2 million rows from a CSV file into SQL Server every night. The current approach inserts rows one at a time using a loop of `INSERT` statements and takes 3 hours. How do you bring this down to minutes?

Replace the row-by-row inserts with `SqlBulkCopy`. It streams data directly into the target table using the TDS bulk insert protocol — the same mechanism behind `BULK INSERT` and `bcp` — bypassing the per-row overhead of individual INSERT statements. A typical implementation reads the CSV with a streaming `IDataReader` (or loads it into a `DataTable` for smaller files) and writes it in one call:

```csharp
using var bulkCopy = new SqlBulkCopy(connection)
{
    DestinationTableName = "Orders",
    BatchSize = 10000,
    BulkCopyTimeout = 600
};
bulkCopy.WriteToServer(dataReader);
```

Set `BatchSize` to control memory usage and lock escalation (e.g., 5,000–50,000 rows per batch). Use the `TableLock` hint for **minimal logging** when the database is in simple or bulk-logged recovery model — this can cut load time by another 50%+. Map columns explicitly with `ColumnMappings` if the CSV schema does not match the table exactly.

**Hint:** Key follow-ups: What are the requirements for minimal logging? (bulk-logged or simple recovery, table lock, empty table or no non-clustered indexes.) What happens if the job fails mid-way — how do you make it resumable? A strong answer mentions wrapping batches in transactions or loading into a staging table first.

**🚩 Red Signal:** Is unaware of `SqlBulkCopy` or continues to advocate for row-by-row inserts "with a larger batch size" as the primary solution.

---

### 2. 🟡 A synchronisation job receives 10,000 product records from an external API and needs to insert new products and update existing ones in a single stored procedure call. Multiple instances of this job can run concurrently. How do you implement this safely?

The standard approach is a `MERGE` statement (UPSERT pattern). Load the incoming data into a staging mechanism (TVP, temp table, or `SqlBulkCopy` to a staging table), then merge:

```sql
MERGE INTO Products AS t
USING @Incoming AS s ON t.ProductId = s.ProductId
WHEN MATCHED THEN
    UPDATE SET t.Name = s.Name, t.Price = s.Price, t.UpdatedAt = GETUTCDATE()
WHEN NOT MATCHED THEN
    INSERT (ProductId, Name, Price, CreatedAt)
    VALUES (s.ProductId, s.Name, s.Price, GETUTCDATE());
```

Under concurrency, `MERGE` alone can still hit race conditions or deadlocks. An alternative that many teams prefer is an explicit `UPDATE` + `INSERT` pattern with proper locking hints:

```sql
UPDATE t WITH (UPDLOCK, HOLDLOCK)
SET t.Name = s.Name, t.Price = s.Price
FROM Products t INNER JOIN @Incoming s ON t.ProductId = s.ProductId;

INSERT INTO Products (ProductId, Name, Price, CreatedAt)
SELECT s.ProductId, s.Name, s.Price, GETUTCDATE()
FROM @Incoming s
WHERE NOT EXISTS (SELECT 1 FROM Products t WITH (UPDLOCK, HOLDLOCK) WHERE t.ProductId = s.ProductId);
```

The `UPDLOCK, HOLDLOCK` hints ensure that the existence check and the subsequent write happen atomically, preventing duplicate key violations when two instances process the same new record simultaneously.

**Hint:** A great answer discusses `MERGE` caveats — known bugs in older SQL Server versions, potential for deadlocks under high concurrency, and the mandatory terminating semicolon. Ask: "Why might a team choose the explicit UPDATE/INSERT pattern over MERGE?"

**🚩 Red Signal:** Writes an UPSERT without any concurrency protection (no `HOLDLOCK`, no `MERGE`), risking duplicate key violations under concurrent execution.

---

### 3. 🟡 A stored procedure needs to accept a list of 500 order IDs from a C# service and return matching records. The current implementation concatenates IDs into a dynamic SQL string: `WHERE Id IN (1,2,3,...)`. What's wrong with this, and what's the better approach?

The dynamic SQL approach has three serious problems: (1) it is vulnerable to SQL injection if the input is not perfectly sanitised, (2) every unique ID list generates a different query string, polluting the plan cache with thousands of single-use plans, and (3) SQL Server has practical limits on the length of `IN` lists and query text.

The better approach is a **Table-Valued Parameter (TVP)**. Define a user-defined table type, then pass the IDs as a structured parameter:

```sql
CREATE TYPE dbo.IdList AS TABLE (Id INT NOT NULL PRIMARY KEY);

-- Stored procedure
CREATE PROCEDURE GetOrdersByIds @Ids dbo.IdList READONLY
AS
    SELECT o.* FROM Orders o
    INNER JOIN @Ids i ON o.Id = i.Id;
```

From C#, pass a `DataTable` or `IEnumerable<SqlDataRecord>` as a `SqlParameter` with `SqlDbType.Structured`. The query plan is reusable, the input is properly parameterised (no injection risk), and it handles thousands of IDs cleanly.

**Hint:** TVPs are read-only inside the procedure. For very large datasets (100K+ rows), `SqlBulkCopy` into a temp table may outperform a TVP. A strong candidate also mentions that `STRING_SPLIT` is a lightweight alternative for simple ID lists in SQL Server 2016+. Follow up: "How does TVP performance change as you scale from 500 to 500,000 IDs?"

**🚩 Red Signal:** Sees nothing wrong with the dynamic SQL approach, or builds comma-separated ID strings and defends it as "standard practice."

---

### 4. 🟡 A developer stores 50,000 intermediate results in a table variable (`@results`) and then joins it to a 2-million-row table. The query takes 40 seconds instead of the expected 2 seconds. The execution plan shows a nested loop join with 50,000 clustered index seeks. Why is this happening, and what's the fix?

Table variables do not maintain statistics. The query optimizer estimates the table variable contains **1 row** regardless of how many rows are actually inserted (this is fixed with `OPTION (RECOMPILE)` or deferred compilation in SQL Server 2019+, but the default behavior still affects many systems). With a 1-row estimate, the optimizer chooses a nested loop join — fine for 1 row, catastrophic for 50,000 rows because it performs 50,000 separate index seeks into the large table.

The fix is to replace the table variable with a **temp table** (`#results`):

```sql
CREATE TABLE #results (OrderId INT PRIMARY KEY, Amount DECIMAL(18,2));
-- ... populate #results ...
SELECT r.*, o.CustomerName
FROM #results r
INNER JOIN Orders o ON r.OrderId = o.OrderId;
```

Temp tables maintain statistics and support full indexes, so the optimizer sees the true row count (50,000) and picks an appropriate join strategy — likely a hash or merge join that processes both sides in a single pass.

| Feature | Temp Table (`#temp`) | Table Variable (`@table`) |
|---|---|---|
| Statistics | Yes — helps query optimizer | No — estimates 1 row |
| Indexes | Full index support | Primary key / unique only (inline) |
| Transaction scope | Participates in transactions | INSERT not rolled back on statement errors |
| Parallelism | Supported | Not supported (before SQL 2019) |
| Recompilation | May trigger recompiles | No recompile |

**Hint:** A strong answer mentions that `OPTION (RECOMPILE)` can force the optimizer to sniff the actual table variable cardinality at execution time, which is a middle-ground fix when refactoring to a temp table is too invasive. Ask: "When would you still prefer a table variable over a temp table?"

**🚩 Red Signal:** Always uses table variables regardless of data size, or cannot explain why the cardinality estimate of 1 leads to a bad plan.
