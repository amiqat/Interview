## SQL vs NoSQL Databases

**Difficulty:** Medium

**Topic:** Database Design, Architecture

### Problem Statement

Explain the differences between SQL and NoSQL databases, their use cases, and when to choose one over the other.

### SQL Databases (Relational)

**Characteristics:**
- Structured data with predefined schema
- Tables with rows and columns
- ACID transactions
- Strong consistency
- SQL query language
- Vertical scaling (traditionally)

**Examples:**
- PostgreSQL
- MySQL
- Oracle
- Microsoft SQL Server
- SQLite

**Strengths:**
- Strong data integrity and consistency
- Complex queries with JOINs
- Mature ecosystem and tools
- Standardized query language (SQL)
- ACID transactions

**Weaknesses:**
- Schema changes can be difficult
- Vertical scaling limits
- Can be slower for simple lookups
- Fixed schema can be restrictive

### NoSQL Databases (Non-Relational)

**Characteristics:**
- Flexible/dynamic schema
- Various data models (document, key-value, graph, column-family)
- Eventual consistency (typically)
- BASE properties
- Horizontal scaling
- Optimized for specific access patterns

### NoSQL Types

**1. Document Databases**
- Store data in JSON-like documents
- Examples: MongoDB, Couchbase, DynamoDB
- Use case: Content management, catalogs, user profiles

**2. Key-Value Stores**
- Simple key-value pairs
- Examples: Redis, Memcached, DynamoDB
- Use case: Caching, session management, real-time data

**3. Column-Family Stores**
- Store data in columns instead of rows
- Examples: Cassandra, HBase, ScyllaDB
- Use case: Time-series data, analytics, IoT

**4. Graph Databases**
- Store data as nodes and relationships
- Examples: Neo4j, Amazon Neptune, ArangoDB
- Use case: Social networks, recommendation engines, fraud detection

### Comparison Table

| Feature | SQL | NoSQL |
|---------|-----|-------|
| Schema | Fixed, predefined | Flexible, dynamic |
| Data Model | Tables with relations | Documents, key-value, graph, etc. |
| Scalability | Vertical (scale up) | Horizontal (scale out) |
| Transactions | ACID guaranteed | Eventually consistent (usually) |
| Query Language | SQL (standardized) | Database-specific APIs |
| Data Integrity | Strong | Flexible |
| Joins | Efficient | Limited or manual |
| Use Cases | Complex transactions, reports | High-volume, simple queries |

### When to Use SQL

**Choose SQL when:**
1. **Complex queries with joins** - Multiple related tables
2. **ACID transactions required** - Financial systems, booking systems
3. **Structured data** - Well-defined schema
4. **Data integrity is critical** - Banking, healthcare
5. **Reporting and analytics** - Complex aggregations

**Examples:**
- E-commerce order management
- Banking systems
- ERP systems
- CRM applications
- Accounting software

### When to Use NoSQL

**Choose NoSQL when:**
1. **Massive scale** - Millions of users, petabytes of data
2. **Flexible schema** - Evolving data structures
3. **High write throughput** - Logging, analytics, IoT
4. **Simple query patterns** - Key-based lookups
5. **Geographic distribution** - Multiple data centers

**Examples:**
- Social media feeds
- Real-time analytics
- IoT sensor data
- Gaming leaderboards
- Content management systems
- Caching layers

### Real-World Examples

**Twitter:**
- Uses MySQL for user data, tweets (core data)
- Uses Redis for caching
- Uses Cassandra for analytics

**Netflix:**
- Uses Cassandra for distributed data
- Uses MySQL for billing
- Uses Redis for caching

**Uber:**
- Uses MySQL for business-critical data
- Uses Redis for geospatial data
- Uses Cassandra for analytics

### Hybrid Approach (Polyglot Persistence)

Many modern applications use both:

```
┌─────────────────────────────────────┐
│         Application Layer           │
└────────┬───────────┬─────────┬──────┘
         │           │         │
    ┌────▼────┐ ┌────▼────┐ ┌─▼──────┐
    │  MySQL  │ │ MongoDB │ │ Redis  │
    │         │ │         │ │        │
    │User Data│ │Catalog  │ │Cache   │
    └─────────┘ └─────────┘ └────────┘
```

**Example Architecture:**
- **PostgreSQL** - User accounts, orders (ACID critical)
- **MongoDB** - Product catalog (flexible schema)
- **Redis** - Session data, caching
- **Elasticsearch** - Full-text search
- **Cassandra** - Time-series analytics

### ACID vs BASE

**ACID (SQL):**
- **A**tomicity - All or nothing
- **C**onsistency - Valid state always
- **I**solation - Transactions don't interfere
- **D**urability - Committed data persists

**BASE (NoSQL):**
- **B**asically **A**vailable - System available most of the time
- **S**oft state - State may change over time
- **E**ventual consistency - Eventually becomes consistent

### Migration Considerations

**SQL to NoSQL:**
- Denormalize data
- Embed related data
- Accept eventual consistency
- Design for access patterns

**NoSQL to SQL:**
- Normalize data structure
- Define relationships
- Implement constraints
- Plan for schema migrations

### Interview Questions

**Q: Can you have transactions in NoSQL?**
A: Some NoSQL databases support limited transactions (e.g., MongoDB multi-document transactions, DynamoDB transactions), but they're typically not as robust as SQL ACID transactions.

**Q: Is NoSQL faster than SQL?**
A: Not necessarily. NoSQL is optimized for specific access patterns and horizontal scaling. For complex queries with joins, SQL can be faster.

**Q: Should I always use NoSQL for big data?**
A: Not always. Modern SQL databases (PostgreSQL, CockroachDB) can handle large scale. Choose based on access patterns, consistency needs, and query complexity.

### Follow-up Questions

1. How would you migrate from SQL to NoSQL?
2. What is the CAP theorem and how does it relate to databases?
3. How do you handle relationships in NoSQL databases?
4. What are the trade-offs of eventual consistency?

### Tags

`database` `sql` `nosql` `architecture` `medium` `design`
