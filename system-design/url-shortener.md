## Design URL Shortener

**Difficulty:** Medium

**Topic:** System Design, Scalability

### Problem Statement

Design a URL shortening service like bit.ly or TinyURL that converts long URLs into short, manageable links.

### Requirements

**Functional Requirements:**
- Given a long URL, generate a shorter unique URL
- When accessing the short URL, redirect to the original long URL
- Links should be customizable (optional)
- Links should have expiration time (optional)

**Non-Functional Requirements:**
- High availability (99.9% uptime)
- Low latency for redirects (< 100ms)
- Scalable to handle millions of URLs
- URLs should be as short as possible

### Capacity Estimation

**Assumptions:**
- 100M new URLs per month
- Read:Write ratio = 100:1 (10B redirects per month)
- URL retention: 5 years
- Average URL size: 500 bytes

**Storage:**
- 100M URLs/month * 500 bytes * 12 months * 5 years = 300 GB

**QPS:**
- Write: 100M / (30 days * 24 hours * 3600 sec) ≈ 40 URLs/sec
- Read: 40 * 100 = 4000 URLs/sec

### High-Level Design

**Components:**
1. **API Gateway** - Entry point for requests
2. **Application Servers** - Business logic
3. **Database** - Store URL mappings
4. **Cache** - Cache popular URLs
5. **Load Balancer** - Distribute traffic

**API Design:**
```
POST /api/v1/shorten
Request: { "long_url": "https://example.com/very/long/url" }
Response: { "short_url": "https://short.ly/abc123" }

GET /abc123
Response: 302 Redirect to long URL
```

### Solution Approach

**1. URL Encoding:**

**Option A: Hash-based**
- Use MD5/SHA-256 hash of long URL
- Take first 6-8 characters
- **Pros:** Simple, deterministic
- **Cons:** Collision handling needed

**Option B: Base62 Encoding**
- Use auto-incrementing ID
- Convert to base62 (a-z, A-Z, 0-9)
- **Pros:** Guaranteed unique, shorter
- **Cons:** Predictable sequence

**Option C: Random Generation + Collision Check**
- Generate random string
- Check if exists, regenerate if collision
- **Pros:** Not predictable, customizable
- **Cons:** Slight performance overhead

**2. Database Schema:**

```sql
CREATE TABLE urls (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    short_code VARCHAR(10) UNIQUE NOT NULL,
    long_url TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP,
    user_id BIGINT,
    click_count INT DEFAULT 0,
    INDEX idx_short_code (short_code),
    INDEX idx_user_id (user_id)
);
```

**3. Caching Strategy:**
- Cache frequently accessed URLs in Redis/Memcached
- Use LRU (Least Recently Used) eviction policy
- Cache for 80/20 rule (20% URLs get 80% traffic)

**4. Database Selection:**
- **SQL (PostgreSQL/MySQL):** ACID guarantees, relations
- **NoSQL (DynamoDB/MongoDB):** High scalability, flexibility

### Architecture Diagram

```
                                    ┌─────────────┐
                                    │Load Balancer│
                                    └──────┬──────┘
                                           │
                    ┌──────────────────────┼──────────────────────┐
                    │                      │                      │
              ┌─────▼─────┐          ┌─────▼─────┐          ┌─────▼─────┐
              │App Server │          │App Server │          │App Server │
              └─────┬─────┘          └─────┬─────┘          └─────┬─────┘
                    │                      │                      │
                    └──────────────────────┼──────────────────────┘
                                           │
                    ┌──────────────────────┼──────────────────────┐
                    │                      │                      │
              ┌─────▼─────┐          ┌─────▼─────┐          ┌─────▼─────┐
              │   Cache   │          │ Database  │          │ Analytics │
              │  (Redis)  │          │  Cluster  │          │  Service  │
              └───────────┘          └───────────┘          └───────────┘
```

### Deep Dive Topics

**1. Handling High Traffic:**
- Use CDN for static content
- Implement rate limiting per user/IP
- Database sharding based on hash of short code
- Read replicas for read-heavy workload

**2. URL Expiration:**
- Background job to clean expired URLs
- Lazy deletion on access
- Return 404 for expired URLs

**3. Analytics:**
- Track clicks, geographic data, referrers
- Use separate analytics database or service
- Async logging to not block redirects

**4. Custom URLs:**
- Check availability before creation
- Reserve popular words/patterns
- Validate custom URL format

### Trade-offs

| Approach | Pros | Cons |
|----------|------|------|
| Hash-based | Deterministic, simple | Collisions, not truly short |
| Auto-increment | Unique, efficient | Predictable, needs counter service |
| Random | Unpredictable, flexible | Rare collisions, overhead |

### Follow-up Questions

1. How would you handle URL deletion or editing?
2. How would you prevent abuse (spam, malicious URLs)?
3. How would you implement analytics without impacting performance?
4. How would you handle database sharding?
5. What happens if the database goes down?
6. How would you implement geographic-based routing?

### Tags

`system-design` `scalability` `database` `caching` `medium`
