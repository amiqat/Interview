# SQL Server

This section covers SQL Server topics commonly explored in .NET backend interviews: indexing strategies, bulk data operations, and query optimization. Questions are framed as real-world scenarios — the kind of problems you encounter in production systems handling millions of rows. Candidates should be comfortable reading execution plans, choosing the right data-loading strategy, and reasoning about concurrency and performance trade-offs.

| File | Topics | Questions |
|---|---|---|
| [Indexing & Performance](indexing-performance.md) | Clustered vs non-clustered indexes, covering indexes, index strategy for mixed workloads, Query Store plan regression | 3 |
| [Bulk Operations](bulk-operations.md) | SqlBulkCopy, UPSERT / MERGE, Table-Valued Parameters, temp tables vs table variables | 4 |
| [Query Optimization](query-optimization.md) | Complex multi-table joins, SARGability, large-data streaming and pagination | 3 |
| [Schema-Based Queries](schema-and-queries.md) | Joins, aggregation, window functions, relational division, index design, UPSERT concurrency — all against a single university schema | 8 |
