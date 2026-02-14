## Design Rate Limiter

**Difficulty:** Medium

**Topic:** System Design, Distributed Systems

### Problem Statement

Design a rate limiter that restricts the number of requests a user or client can make to an API within a specified time window.

### Requirements

**Functional Requirements:**
- Limit requests based on user ID, IP address, or API key
- Support different rate limiting rules (requests per second/minute/hour)
- Return appropriate error when limit is exceeded
- Support different limits for different users/tiers

**Non-Functional Requirements:**
- Low latency (should not slow down API requests significantly)
- Accurate (no false positives)
- Scalable across multiple servers
- Fault tolerant

### High-Level Design

**Components:**
1. **Rate Limiter Middleware** - Intercepts requests
2. **Rules Cache** - Stores rate limiting rules
3. **Counter Storage** - Tracks request counts (Redis)
4. **Configuration Service** - Manages rate limit rules

### Solution Approach

**Algorithm Options:**

**1. Token Bucket**
- Each user has a bucket with tokens
- Tokens refill at a constant rate
- Request consumes one token
- **Pros:** Smooth traffic, allows bursts
- **Cons:** Complex implementation

**2. Leaky Bucket**
- Requests enter a queue (bucket)
- Processed at constant rate
- Overflow requests are dropped
- **Pros:** Smooth output rate
- **Cons:** Can reject requests even if system has capacity

**3. Fixed Window Counter**
- Count requests in fixed time windows (e.g., per minute)
- Reset counter at window boundary
- **Pros:** Simple, memory efficient
- **Cons:** Burst at window boundaries

**4. Sliding Window Log**
- Store timestamp of each request
- Count requests in sliding time window
- **Pros:** Very accurate
- **Cons:** High memory usage

**5. Sliding Window Counter (Recommended)**
- Hybrid of fixed window and sliding window log
- Use current and previous window counts
- **Pros:** Smooth, memory efficient, accurate
- **Cons:** Slightly complex

### Implementation

**Sliding Window Counter Algorithm:**

```python
import time
import redis

class RateLimiter:
    def __init__(self, redis_client, max_requests, window_seconds):
        self.redis = redis_client
        self.max_requests = max_requests
        self.window_seconds = window_seconds
    
    def is_allowed(self, user_id):
        """
        Check if user is allowed to make a request.
        
        Args:
            user_id: Identifier for the user
            
        Returns:
            True if allowed, False if rate limit exceeded
        """
        current_time = time.time()
        current_window = int(current_time / self.window_seconds)
        previous_window = current_window - 1
        
        # Keys for current and previous windows
        current_key = f"rate_limit:{user_id}:{current_window}"
        previous_key = f"rate_limit:{user_id}:{previous_window}"
        
        # Get counts
        current_count = int(self.redis.get(current_key) or 0)
        previous_count = int(self.redis.get(previous_key) or 0)
        
        # Calculate sliding window count
        elapsed_time_in_window = current_time % self.window_seconds
        weight = (self.window_seconds - elapsed_time_in_window) / self.window_seconds
        estimated_count = previous_count * weight + current_count
        
        if estimated_count >= self.max_requests:
            return False
        
        # Increment counter
        pipe = self.redis.pipeline()
        pipe.incr(current_key)
        pipe.expire(current_key, self.window_seconds * 2)  # Keep for 2 windows
        pipe.execute()
        
        return True
```

**Simple Fixed Window Implementation:**

```python
def is_allowed_fixed_window(user_id, max_requests, window_seconds):
    """
    Simple fixed window rate limiter.
    
    Args:
        user_id: Identifier for the user
        max_requests: Maximum requests allowed in window
        window_seconds: Time window in seconds
        
    Returns:
        True if allowed, False if rate limit exceeded
    """
    current_window = int(time.time() / window_seconds)
    key = f"rate_limit:{user_id}:{current_window}"
    
    # Use Redis INCR for atomic increment
    count = redis_client.incr(key)
    
    if count == 1:
        # First request in window, set expiration
        redis_client.expire(key, window_seconds)
    
    return count <= max_requests
```

### Architecture

**Distributed Rate Limiter:**

```
┌─────────────┐         ┌─────────────┐         ┌─────────────┐
│ API Server  │         │ API Server  │         │ API Server  │
│             │         │             │         │             │
│ Rate Limiter│         │ Rate Limiter│         │ Rate Limiter│
└──────┬──────┘         └──────┬──────┘         └──────┬──────┘
       │                       │                       │
       └───────────────────────┼───────────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │  Redis Cluster      │
                    │  (Shared Counter)   │
                    └─────────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │ Rules Configuration │
                    │     Database        │
                    └─────────────────────┘
```

### Data Schema

**Redis Keys:**
```
rate_limit:{user_id}:{window_timestamp} -> count
rate_limit:rules:{user_id} -> {max_requests, window_seconds}
```

**Rules Configuration (Database):**
```sql
CREATE TABLE rate_limit_rules (
    id BIGINT PRIMARY KEY,
    user_id VARCHAR(255),
    api_endpoint VARCHAR(255),
    max_requests INT NOT NULL,
    window_seconds INT NOT NULL,
    tier VARCHAR(50),  -- free, premium, enterprise
    created_at TIMESTAMP,
    INDEX idx_user_id (user_id)
);
```

### Response Format

**When Rate Limited:**
```
HTTP 429 Too Many Requests

Headers:
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1623456789

Body:
{
    "error": "Rate limit exceeded",
    "message": "You have exceeded 100 requests per hour",
    "retry_after": 3600
}
```

### Advanced Features

**1. Different Tiers:**
```python
tier_limits = {
    "free": {"max_requests": 100, "window": 3600},
    "premium": {"max_requests": 1000, "window": 3600},
    "enterprise": {"max_requests": 10000, "window": 3600}
}
```

**2. Per-Endpoint Limits:**
- Different limits for different API endpoints
- More restrictive limits for expensive operations

**3. Distributed Rate Limiting:**
- Use Redis cluster for shared state
- Consistent hashing for data distribution
- Handle Redis failures gracefully

### Trade-offs

| Algorithm | Accuracy | Memory | Burst Handling |
|-----------|----------|--------|----------------|
| Token Bucket | High | Low | Yes |
| Leaky Bucket | High | Medium | Limited |
| Fixed Window | Medium | Low | Poor (edge burst) |
| Sliding Log | Very High | High | Good |
| Sliding Window | High | Low | Good |

### Follow-up Questions

1. How would you handle rate limiting for distributed systems?
2. What if Redis goes down? How do you ensure availability?
3. How would you implement rate limiting at the API gateway level?
4. How would you handle rate limiting for websocket connections?
5. How would you implement geographic-based rate limiting?
6. How would you handle DDoS attacks?

### Tags

`system-design` `rate-limiting` `redis` `distributed-systems` `medium`
