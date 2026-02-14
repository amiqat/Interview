# SQL Server — Senior Interview Questions

## Complex Queries

### 1. Explain CTEs (Common Table Expressions) and recursive CTEs.

**Hint:** CTE is a named temporary result set defined with `WITH`. Recursive CTE references itself — useful for hierarchies (org charts, tree structures). Must have an anchor member and a recursive member with a termination condition.

**🚩 Red Signal:** Cannot write a basic CTE or confuses it with temp tables. Doesn't mention termination condition for recursive CTEs.

---

### 2. What are window functions and when do you use them?

**Hint:** `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()`, `LEAD()`, `LAG()`, `SUM() OVER(...)`. Operate on a set of rows related to the current row without collapsing them. Use for running totals, rankings, pagination, gap detection.

**🚩 Red Signal:** Uses self-joins or correlated subqueries for problems easily solved by window functions.

---

### 3. How do you de-duplicate rows efficiently?

**Hint:** Use `ROW_NUMBER() OVER (PARTITION BY ... ORDER BY ...)` in a CTE, then delete where `RowNum > 1`. Alternatively, `SELECT DISTINCT INTO` a new table. Consider which row to keep and why.

**🚩 Red Signal:** Uses cursors to loop through duplicates or has no strategy for choosing which row to keep.

---

## Indexes

### 4. What is the difference between clustered and non-clustered indexes?

**Hint:** Clustered index defines the physical order of data (one per table, usually PK). Non-clustered index is a separate B-tree with pointers back to the data. Include columns via `INCLUDE` to create covering indexes.

**🚩 Red Signal:** Thinks a table can have multiple clustered indexes or doesn't understand what a covering index is.

---

### 5. When would you use a filtered index?

**Hint:** Index with a `WHERE` clause — indexes only a subset of rows. Great for columns with many NULLs or skewed distributions (e.g., `WHERE IsActive = 1`). Smaller index, faster lookups for that subset.

**🚩 Red Signal:** Creates full indexes on columns where only a small subset of values is queried.

---

### 6. How do you identify and fix missing or unused indexes?

**Hint:** Missing: `sys.dm_db_missing_index_details`, Query Store, execution plans with index suggestions. Unused: `sys.dm_db_index_usage_stats` — look for indexes with zero seeks/scans. Drop unused indexes to reduce write overhead.

**🚩 Red Signal:** Adds indexes based on guesswork without checking execution plans or DMVs.

---

### 7. What is index fragmentation and how do you address it?

**Hint:** Logical fragmentation (out-of-order pages) and physical fragmentation. Check with `sys.dm_db_index_physical_stats`. Rebuild (`ALTER INDEX REBUILD`) for >30% fragmentation, reorganize for 10-30%. Consider fill factor for insert-heavy tables.

**🚩 Red Signal:** Rebuilds all indexes nightly without checking fragmentation levels, or doesn't know the difference between rebuild and reorganize.

---

## Large Data & Bulk Operations

### 8. How does `SqlBulkCopy` work and when should you use it?

**Hint:** Streams data directly into the table using TDS bulk insert protocol. Minimal logging with simple/bulk-logged recovery model. Set `BatchSize` to control memory and lock duration. Use `BulkCopyTimeout` for large loads. Much faster than row-by-row `INSERT`.

**🚩 Red Signal:** Inserts millions of rows with individual `INSERT` statements or Entity Framework `Add`/`SaveChanges` in a loop.

---

### 9. How do you perform an UPSERT in SQL Server?

**Hint:** Use `MERGE ... WHEN MATCHED THEN UPDATE WHEN NOT MATCHED THEN INSERT`. Alternatively, use `INSERT ... ON CONFLICT`-style patterns with `IF EXISTS` + `UPDATE` / `INSERT` (or `UPDATE` + `@@ROWCOUNT` check + `INSERT`). `MERGE` has known concurrency bugs — use with `HOLDLOCK` or prefer the two-statement pattern with proper locking.

**🚩 Red Signal:** Uses `MERGE` without locking hints or is unaware of the concurrency issues with `MERGE`.

---

### 10. What are temp tables vs table variables and when do you use each?

**Hint:** Temp tables (`#temp`): stored in `tempdb`, support indexes, statistics, and parallelism. Table variables (`@table`): estimated 1 row by optimizer (pre-2019), no parallel plans, better for small datasets. Temp tables are generally better for large datasets due to statistics.

**🚩 Red Signal:** Always uses table variables for large datasets or thinks table variables live in memory (they don't — both are in `tempdb`).

---

### 11. What are Table-Valued Parameters (TVP)?

**Hint:** User-defined table types passed as parameters to stored procedures/functions. Replaces comma-delimited strings or XML for passing collections. Strongly typed, supports multiple columns. Readonly in the procedure. Combine with `MERGE` or `INSERT ... SELECT` for bulk operations.

**🚩 Red Signal:** Passes large CSV strings and parses them inside stored procedures instead of using TVPs.

---

### 12. How do you handle large data reads without blocking or memory issues?

**Hint:** Use `NOLOCK` / `READ UNCOMMITTED` (accept dirty reads) or `SNAPSHOT` isolation for consistent reads without blocking. Stream results with `DataReader` instead of loading into `DataSet`. Paginate with `OFFSET/FETCH`. Consider read replicas for reporting.

**🚩 Red Signal:** Reads millions of rows into a `DataTable` in memory, or uses `NOLOCK` everywhere without understanding dirty read implications.

---

### 13. What is query plan caching and parameter sniffing?

**Hint:** SQL Server caches execution plans keyed by query text/parameters. Parameter sniffing: plan compiled for first parameter values may be suboptimal for others. Mitigate with `OPTIMIZE FOR`, `RECOMPILE`, plan guides, or Query Store forced plans.

**🚩 Red Signal:** Doesn't know what parameter sniffing is, or always uses `RECOMPILE` without understanding the compilation cost.
