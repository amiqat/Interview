# Indexing & Performance

---

### 1. 🟡 Your `Orders` table has 10 million rows with a clustered index on `Id`. A critical query filters by `CustomerId` and sorts by `OrderDate`, but the execution plan shows a full clustered index scan. What do you do to fix this?

The query cannot use the clustered index on `Id` because it filters on `CustomerId` and sorts on `OrderDate` — neither column is the clustering key. You need to create a non-clustered composite index on `(CustomerId, OrderDate)`. The key order matters: `CustomerId` first for the equality filter, `OrderDate` second for the sort, so SQL Server can seek directly to matching rows and return them already sorted without an extra Sort operator.

If the query also selects other columns (e.g., `Amount`, `Status`), SQL Server will perform a key lookup back to the clustered index for each matching row, which is expensive at scale. To eliminate this, add those columns with `INCLUDE`:

```sql
CREATE NONCLUSTERED INDEX IX_Orders_Customer_Date
ON Orders (CustomerId, OrderDate)
INCLUDE (Amount, Status);
```

This makes it a **covering index** — the entire query is satisfied from the index leaf pages without touching the base table.

**Hint:** A strong answer explains why key column order matters (equality columns before range/sort columns), articulates the cost of key lookups, and mentions checking the actual execution plan — not just the estimated one — to verify the scan is eliminated. Follow up: "What happens if you reverse the key order to `(OrderDate, CustomerId)`?"

**🚩 Red Signal:** Suggests adding two separate single-column indexes (one on `CustomerId`, one on `OrderDate`) and expects SQL Server to combine them efficiently, or does not know that a table can have only one clustered index.

---

### 2. 🔴 You inherit a table with 5 million rows that serves both a high-throughput order-entry API (hundreds of inserts/updates per second) and a reporting dashboard that runs complex analytical queries. The table currently has 12 non-clustered indexes, and write latency is climbing. How do you approach re-evaluating the indexing strategy?

Start by measuring, not guessing. Query `sys.dm_db_index_usage_stats` to find indexes with high write cost (`user_updates`) but low or zero read benefit (`user_seeks`, `user_scans`). Drop any index that is never used for reads — it is pure write overhead. Next, examine the most expensive queries via Query Store or `sys.dm_exec_query_stats` and cross-reference with `sys.dm_db_missing_index_details` for hints (treat these as suggestions, not instructions).

Consolidate overlapping indexes: if you have indexes on `(A)`, `(A, B)`, and `(A, B, C)`, the first is redundant. Prefer narrow composite indexes with `INCLUDE` columns for covering. For the analytical dashboard, consider a non-clustered **columnstore index** — it dramatically accelerates aggregation and scan-heavy queries without adding many B-tree indexes.

Finally, schedule index maintenance: rebuild indexes above 30% fragmentation, reorganise between 10–30%, and review the strategy quarterly as query patterns evolve.

**Hint:** A senior candidate should discuss write amplification (every index adds an extra write on INSERT/UPDATE/DELETE), the trade-off between read and write performance, and the use of filtered indexes for queries that target a subset of rows (e.g., `WHERE Status = 'Active'`). Ask: "How do you decide when to rebuild vs reorganise an index?"

**🚩 Red Signal:** Adds indexes to every filtered column "just in case," never checks `sys.dm_db_index_usage_stats`, or is unaware that unused indexes silently hurt write performance.

---

### 3. 🟡 A report query that used to run in 2 seconds now takes 30 seconds. Nothing in the query text has changed, but a statistics update ran overnight. How do you diagnose and fix this plan regression?

The most likely cause is a **plan regression** — the statistics update caused the query optimizer to generate a new execution plan that is worse than the previous one (e.g., switching from an index seek to a hash join due to updated cardinality estimates). Open **Query Store** and look at the query's plan history: you will see the old fast plan and the new slow plan side by side.

In Query Store's "Regressed Queries" view, identify the regressed query, compare the two plans, and **force the known-good plan**:

```sql
EXEC sp_query_store_force_plan @query_id = 42, @plan_id = 7;
```

This tells SQL Server to always use the specified plan for that query, regardless of future statistics changes. For a longer-term fix, investigate *why* the new plan is worse — perhaps the statistics are skewed, a parameter-sniffing issue is at play, or the query would benefit from `OPTION (RECOMPILE)` or a plan guide.

Ensure Query Store is in `READ_WRITE` mode and that the `STALE_QUERY_THRESHOLD_DAYS` cleanup policy retains enough history to catch regressions. Monitor the `sys.query_store_runtime_stats` view to set up alerts for future regressions.

**Hint:** A strong answer covers what Query Store captures (plans, runtime stats, wait stats), the difference between forcing a plan and fixing the root cause, and how parameter sniffing can interact with statistics updates. Follow up: "What if Query Store is not enabled — how would you diagnose this?"

**🚩 Red Signal:** Relies solely on `sp_who2` or Activity Monitor for production troubleshooting, or has no strategy for handling plan regressions beyond restarting the SQL Server service.
