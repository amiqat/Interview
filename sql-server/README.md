# SQL Server

Questions on complex queries, indexing strategies, handling large data, bulk operations, temp tables, and Table-Valued Parameters (TVP).

---

### 1. Explain the difference between clustered and non-clustered indexes. When would you choose one over the other?

A **clustered index** determines the physical order of data in the table — there can be only one per table (typically the primary key). A **non-clustered index** is a separate B-tree structure with pointers (row locators) back to the data pages.

Choose a clustered index for columns used in range scans (`BETWEEN`, `ORDER BY`) and the primary lookup key. Use non-clustered indexes for frequently filtered or joined columns that are not the primary key.

**Hint:** Look for understanding of covering indexes (`INCLUDE`), the concept of a key lookup (bookmark lookup) and its cost, and filtered indexes for partial data.

**🚩 Red Signal:** Believes you can have multiple clustered indexes on a table, or confuses indexes with constraints.

---

### 2. What is a covering index and how does it eliminate key lookups?

A covering index contains all columns needed by a query — either as key columns or via `INCLUDE` columns. When the query can be satisfied entirely from the index, SQL Server avoids the expensive key lookup (or RID lookup) back to the clustered index/heap.

**Hint:** The candidate should discuss the trade-off: wider indexes consume more storage and slow down writes. They should know how to read an execution plan to spot key lookups.

**🚩 Red Signal:** Cannot explain what a key lookup is or why it is expensive.

---

### 3. How do you approach indexing a table with millions of rows and mixed read/write workloads?

- Analyse the most frequent and expensive queries using the Query Store or DMVs (`sys.dm_exec_query_stats`).
- Use the missing index DMVs (`sys.dm_db_missing_index_details`) as hints, not gospel.
- Prefer narrow indexes; use `INCLUDE` for covering.
- Monitor index usage with `sys.dm_db_index_usage_stats` and drop unused indexes.
- Consider filtered indexes and columnstore indexes for analytical queries.

**Hint:** A senior candidate should mention the write amplification cost of too many indexes and the importance of index maintenance (rebuild/reorganize based on fragmentation).

**🚩 Red Signal:** Adds indexes to every column "just in case" or never monitors index usage.

---

### 4. Explain `SqlBulkCopy` and how you use it to perform high-performance inserts into SQL Server.

`SqlBulkCopy` streams data from a `DataTable`, `IDataReader`, or `DataRow[]` directly into a SQL Server table using the TDS bulk insert protocol — the same mechanism as `BULK INSERT` and `bcp`. It bypasses the normal INSERT statement overhead and can insert millions of rows per second.

**Hint:** Key options: `BatchSize` (controls memory and lock escalation), `BulkCopyTimeout`, `TableLock` hint for minimal logging, and column mappings for schema mismatches. Understanding of minimal logging requirements (simple/bulk-logged recovery model, table lock, no non-clustered indexes or empty table).

**🚩 Red Signal:** Uses a loop of individual `INSERT` statements for loading millions of rows, or is unaware of `SqlBulkCopy`.

---

### 5. How do you implement an UPSERT (INSERT or UPDATE) pattern in SQL Server?

The standard pattern is `MERGE`:

```sql
MERGE INTO Target AS t
USING Source AS s ON t.Key = s.Key
WHEN MATCHED THEN UPDATE SET t.Col = s.Col
WHEN NOT MATCHED THEN INSERT (Key, Col) VALUES (s.Key, s.Col);
```

Alternatively, use `INSERT ... ON CONFLICT`-style with an explicit `IF EXISTS` / `UPDATE` + `INSERT` pattern with proper locking (`UPDLOCK, HOLDLOCK`) to prevent race conditions.

**Hint:** A great answer discusses `MERGE` caveats (bugs in older SQL Server versions, the requirement for a terminating semicolon, and potential for deadlocks) and why some teams prefer the explicit `UPDATE`/`INSERT` pattern with `HOLDLOCK`.

**🚩 Red Signal:** Writes an UPSERT without any concurrency protection (no `HOLDLOCK` or `MERGE`), risking duplicate key violations under load.

---

### 6. What are Table-Valued Parameters (TVP) and when should you use them?

TVPs allow you to pass a structured table of data to a stored procedure or parameterised query. You define a `CREATE TYPE ... AS TABLE`, then pass a `DataTable` or `IEnumerable<SqlDataRecord>` from C# as a `SqlParameter` with `SqlDbType.Structured`.

Use cases: passing a list of IDs for an `IN` clause, batch operations, and replacing XML/JSON parameter hacks.

**Hint:** TVPs are read-only in the stored procedure. For very large datasets (100K+ rows), `SqlBulkCopy` to a staging table may outperform TVPs.

**🚩 Red Signal:** Builds dynamic SQL with string-concatenated IDs (e.g., `WHERE Id IN (1,2,3,...)`) instead of using a TVP or temp table.

---

### 7. Compare temp tables (`#temp`) vs table variables (`@table`). When do you choose each?

| Feature | Temp Table (`#temp`) | Table Variable (`@table`) |
|---|---|---|
| Statistics | Yes — helps query optimizer | No — estimates 1 row |
| Indexes | Full index support | Primary key / unique only (inline) |
| Transaction scope | Participates in transactions | INSERT not rolled back on error* |
| Parallelism | Supported | Not supported (before SQL 2019) |
| Recompilation | May trigger recompiles | No recompile |

*Table variable INSERTs do not roll back on statement-level errors, but do on explicit `ROLLBACK`.

**Hint:** Use temp tables for large result sets (thousands of rows) where statistics matter. Use table variables for small, well-known result sets where you want to avoid recompilation.

**🚩 Red Signal:** Always uses table variables regardless of data size, leading to terrible query plans due to cardinality estimate of 1.

---

### 8. How do you write and optimise a complex query that joins 5+ tables with large data volumes?

- Start with the correct JOIN types (INNER, LEFT) and ensure proper ON clauses.
- Examine the execution plan: look for table scans, high-cost operators, key lookups, and hash joins on small tables.
- Ensure join columns are indexed.
- Use CTEs or derived tables for readability, but understand they are inlined (not materialised).
- Consider breaking into staged temp tables if the optimizer produces a bad plan.
- Use `OPTION (RECOMPILE)` or plan guides for parameter-sensitive queries.

**Hint:** Look for understanding of SARGability — predicates like `WHERE YEAR(DateCol) = 2024` prevent index usage; rewrite as `WHERE DateCol >= '2024-01-01' AND DateCol < '2025-01-01'`.

**🚩 Red Signal:** Writes complex queries without ever looking at the execution plan.

---

### 9. What is the Query Store and how do you use it to troubleshoot performance regressions?

Query Store captures query plans, execution statistics, and wait stats at the query level. It allows you to:
- Identify regressed queries by comparing recent vs historical plans.
- Force a known-good plan for a regressed query.
- Analyse top resource-consuming queries.

**Hint:** A strong answer covers the difference between `READ_WRITE` and `READ_ONLY` modes, the cleanup policy (`STALE_QUERY_THRESHOLD_DAYS`), and how plan forcing avoids parameter-sniffing regressions.

**🚩 Red Signal:** Relies solely on `sp_who2` or Activity Monitor for production troubleshooting.

---

### 10. How do you handle processing or exporting millions of rows without running out of memory?

- **Server-side cursor / streaming:** Use `SqlDataReader` with `CommandBehavior.SequentialAccess` to stream rows without buffering the entire result set.
- **Batching:** Process in chunks using `OFFSET/FETCH` or keyset pagination (`WHERE Id > @lastId ORDER BY Id`).
- **Bulk export:** `bcp` utility or SSIS for server-side exports.
- **Columnstore indexes** for fast aggregation queries on large tables.

**Hint:** Keyset pagination is far more efficient than `OFFSET/FETCH` for large offsets because it can seek directly to the starting key.

**🚩 Red Signal:** Calls `.ToList()` on a multi-million-row query, loading everything into memory.
