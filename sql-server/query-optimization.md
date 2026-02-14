# Query Optimization

---

### 1. 🔴 A query joining 5 tables runs in under a second in dev (1,000 rows per table) but takes 30+ seconds in production (millions of rows per table). The query text and indexes are identical in both environments. Walk through how you would diagnose and optimize this.

Start by pulling the **actual execution plan** in production (not the estimated plan — actual row counts matter). Compare it with the dev plan. Common findings: the optimizer chose a nested loop join that works fine for 1,000 rows but is catastrophic for millions (it should be a hash or merge join), or a key lookup that returns 5 rows in dev returns 500,000 in production.

Check join columns for missing or mismatched indexes. Ensure every ON clause column has a supporting index on the inner side of the join. Look for implicit conversions (e.g., joining `NVARCHAR` to `VARCHAR`) that silently prevent index usage.

If the optimizer is producing a bad plan due to parameter sniffing (the plan was compiled with atypical parameter values), use `OPTION (RECOMPILE)` to force fresh compilation, or use `OPTIMIZE FOR UNKNOWN` for a more generic plan. For genuinely complex queries where the optimizer struggles, consider breaking the query into staged temp tables — materialise the result of the first 2–3 joins, then join the temp table to the remaining tables. This gives the optimizer accurate statistics at each stage.

Finally, review SARGability of all WHERE and JOIN predicates, check for missing `WHERE` clause filters that could reduce intermediate result sets, and ensure statistics are up to date.

**Hint:** A senior candidate should discuss reading plan operators (hash join vs merge join vs nested loop and when each is appropriate), identifying the most expensive operator, and the cost of spills to tempdb. Follow up: "The plan shows a hash join with a spill warning — what does that mean and how do you fix it?"

**🚩 Red Signal:** Writes complex queries without ever examining the execution plan, or "optimizes" by adding `NOLOCK` everywhere without understanding the consequences.

---

### 2. 🟡 A developer writes `WHERE YEAR(OrderDate) = 2024` on a table with an index on `OrderDate`. The query scans the entire index instead of seeking. Why does this happen, and how do you fix it?

Wrapping a column in a function — `YEAR(OrderDate)` — makes the predicate **non-SARGable** (Search ARGument able). The index is a B-tree sorted by raw `OrderDate` values; SQL Server cannot seek into it using the output of `YEAR()` because the function must be evaluated for every row first. The result is an index scan (or table scan) instead of an efficient index seek.

The fix is to rewrite the predicate as a range that operates directly on the column:

```sql
-- ❌ Non-SARGable
WHERE YEAR(OrderDate) = 2024

-- ✅ SARGable
WHERE OrderDate >= '2024-01-01' AND OrderDate < '2025-01-01'
```

This allows SQL Server to seek directly to the start of 2024 data in the index and read only the relevant pages. The same principle applies to any function on an indexed column: `CAST()`, `CONVERT()`, `ISNULL()`, `UPPER()`, string concatenation, and arithmetic operations all break SARGability.

**Hint:** A strong answer generalises beyond this specific example — any expression that transforms the column before comparison prevents a seek. Ask: "What about `WHERE ISNULL(Status, 'Unknown') = 'Active'` — is that SARGable? How would you rewrite it?" Also probe for awareness of computed columns with persisted indexes as an alternative when the function cannot be avoided.

**🚩 Red Signal:** Does not know the term SARGable, or believes adding an index alone is sufficient without considering how predicates are written.

---

### 3. 🔴 You need to export 8 million rows from a production SQL Server table to a downstream system nightly. The previous developer loaded all rows into a `List<T>` in C# and then wrote to a file — the service crashes with an `OutOfMemoryException` after consuming 4 GB of RAM. How do you redesign this?

Never materialise the entire result set in memory. Use `SqlDataReader` with `CommandBehavior.SequentialAccess` to stream rows one at a time from the server, writing each row (or small batch) directly to the output file or downstream API as it is read:

```csharp
using var reader = await cmd.ExecuteReaderAsync(CommandBehavior.SequentialAccess);
using var writer = new StreamWriter("export.csv");
while (await reader.ReadAsync())
{
    writer.WriteLine($"{reader.GetInt32(0)},{reader.GetString(1)}");
}
```

Memory usage stays constant regardless of table size because only one row (or one buffer) is in memory at any time.

For even better throughput, consider server-side export using the `bcp` utility or SSIS, which avoid pulling data through the application layer entirely. If the query itself is slow on 8 million rows, add a columnstore index for fast full-table scans, or paginate with **keyset pagination** (`WHERE Id > @lastId ORDER BY Id FETCH NEXT 10000 ROWS ONLY`) rather than `OFFSET/FETCH`, which degrades at large offsets because it must scan and discard all skipped rows.

For incremental exports, track a high-watermark (e.g., `LastModifiedDate` or a `rowversion` column) so you only export changed rows each night instead of the full table.

**Hint:** A strong answer distinguishes keyset pagination from offset pagination and explains why offset degrades (it re-scans skipped rows). Also probe: "What if the downstream system needs the data as JSON — how do you stream JSON for 8 million rows without buffering?" Look for awareness of `Utf8JsonWriter` or streaming serialisation.

**🚩 Red Signal:** Calls `.ToList()` or `.ToArray()` on a multi-million-row query, or sees nothing wrong with loading entire result sets into memory.
