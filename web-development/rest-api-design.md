## REST API Design Best Practices

**Difficulty:** Medium

**Topic:** Web Development, API Design

### Problem Statement

Explain REST API design principles and best practices. Design a RESTful API for a blog application.

### REST Principles

**REST (Representational State Transfer)** is an architectural style for designing networked applications.

**Key Principles:**

1. **Client-Server Architecture** - Separation of concerns
2. **Stateless** - Each request contains all information needed
3. **Cacheable** - Responses should be cacheable when appropriate
4. **Uniform Interface** - Standard methods (GET, POST, PUT, DELETE)
5. **Layered System** - Client doesn't need to know about intermediate layers
6. **Code on Demand** (optional) - Server can send executable code

### HTTP Methods

| Method | Purpose | Idempotent | Safe |
|--------|---------|------------|------|
| GET | Retrieve resource(s) | Yes | Yes |
| POST | Create new resource | No | No |
| PUT | Update/replace resource | Yes | No |
| PATCH | Partial update | No | No |
| DELETE | Remove resource | Yes | No |

**Idempotent:** Multiple identical requests have the same effect as a single request
**Safe:** Does not modify resources

### Best Practices

**1. Use Nouns, Not Verbs**

❌ Bad:
```
POST /createUser
GET /getUser/123
POST /updateUser/123
```

✅ Good:
```
POST /users
GET /users/123
PUT /users/123
```

**2. Use Plural Nouns**

❌ Bad: `/user/123`
✅ Good: `/users/123`

**3. Hierarchical Relationships**

```
GET /users/123/posts          # Get all posts by user 123
GET /users/123/posts/456      # Get post 456 by user 123
POST /users/123/posts         # Create a post for user 123
```

**4. Filtering, Sorting, Pagination**

```
GET /users?role=admin                    # Filter
GET /posts?sort=created_at&order=desc   # Sort
GET /posts?page=2&limit=20              # Pagination
GET /posts?author=john&status=published # Multiple filters
```

**5. HTTP Status Codes**

**Success:**
- `200 OK` - Successful GET, PUT, PATCH, DELETE
- `201 Created` - Successful POST
- `204 No Content` - Successful DELETE (no response body)

**Client Errors:**
- `400 Bad Request` - Invalid request data
- `401 Unauthorized` - Authentication required
- `403 Forbidden` - Authenticated but not authorized
- `404 Not Found` - Resource doesn't exist
- `422 Unprocessable Entity` - Validation errors

**Server Errors:**
- `500 Internal Server Error` - Server error
- `503 Service Unavailable` - Server overloaded/down

### Example: Blog API Design

**Resources:**
- Users
- Posts
- Comments
- Tags

**Endpoints:**

```
# Users
GET    /api/v1/users              # List all users
GET    /api/v1/users/:id          # Get specific user
POST   /api/v1/users              # Create user
PUT    /api/v1/users/:id          # Update user
DELETE /api/v1/users/:id          # Delete user

# Posts
GET    /api/v1/posts              # List all posts
GET    /api/v1/posts/:id          # Get specific post
POST   /api/v1/posts              # Create post
PUT    /api/v1/posts/:id          # Update post
DELETE /api/v1/posts/:id          # Delete post

# Comments (nested under posts)
GET    /api/v1/posts/:id/comments           # List post comments
POST   /api/v1/posts/:id/comments           # Create comment
PUT    /api/v1/posts/:id/comments/:commentId  # Update comment
DELETE /api/v1/posts/:id/comments/:commentId  # Delete comment

# Search and Filter
GET    /api/v1/posts?author=123&tag=tech&status=published
GET    /api/v1/posts?search=javascript&sort=created_at&order=desc
```

### Request/Response Examples

**Create a Post:**

Request:
```http
POST /api/v1/posts
Content-Type: application/json
Authorization: Bearer <token>

{
  "title": "Getting Started with REST APIs",
  "content": "REST APIs are...",
  "tags": ["api", "rest", "tutorial"],
  "status": "published"
}
```

Response:
```http
HTTP/1.1 201 Created
Location: /api/v1/posts/789
Content-Type: application/json

{
  "id": 789,
  "title": "Getting Started with REST APIs",
  "content": "REST APIs are...",
  "tags": ["api", "rest", "tutorial"],
  "status": "published",
  "author": {
    "id": 123,
    "name": "John Doe"
  },
  "created_at": "2024-01-15T10:30:00Z",
  "updated_at": "2024-01-15T10:30:00Z"
}
```

**Error Response:**

```http
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/json

{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid input data",
    "details": [
      {
        "field": "title",
        "message": "Title is required"
      },
      {
        "field": "content",
        "message": "Content must be at least 10 characters"
      }
    ]
  }
}
```

### Versioning

**Options:**

1. **URL Path** (recommended):
   ```
   /api/v1/users
   /api/v2/users
   ```

2. **Query Parameter**:
   ```
   /api/users?version=1
   ```

3. **Header**:
   ```
   Accept: application/vnd.myapi.v1+json
   ```

### Authentication

**Common Methods:**

1. **API Keys** - Simple, less secure
   ```
   Authorization: ApiKey abc123
   ```

2. **Bearer Tokens** (JWT) - Most common
   ```
   Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
   ```

3. **OAuth 2.0** - Industry standard
   ```
   Authorization: Bearer <access_token>
   ```

### Rate Limiting

Include rate limit information in response headers:

```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1623456789
```

### Documentation

Use tools like:
- **Swagger/OpenAPI** - Interactive documentation
- **Postman Collections** - API examples
- **API Blueprint** - Markdown-based documentation

### Common Pitfalls

1. **Using verbs in URLs** - Use HTTP methods instead
2. **Not using HTTP status codes correctly**
3. **Not versioning your API**
4. **Returning inconsistent response formats**
5. **Not implementing proper error handling**
6. **Ignoring security (HTTPS, authentication)**
7. **Not documenting the API**

### Follow-up Questions

1. How would you handle long-running operations in a REST API?
2. What's the difference between PUT and PATCH?
3. How would you implement pagination for large datasets?
4. How do you handle API versioning when making breaking changes?
5. What's the difference between REST and GraphQL?

### Tags

`web-development` `rest-api` `api-design` `http` `medium`
