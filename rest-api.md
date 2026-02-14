# REST API, Swagger & Authentication — Senior Interview Questions

> **Time:** ~15 min · **Pick:** 4-6 questions · **Start with** 🟢 then go deeper

---

## REST API Conventions

### 🟢 1. What are the key REST constraints and how do you apply them in .NET?

**Hint:** Stateless, client-server, cacheable, uniform interface, layered system. In .NET:
- Use HTTP verbs correctly (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`)
- Return proper status codes
- Use resource-based URIs — nouns not verbs (`/api/orders`, NOT `/api/getOrders`)
- Support content negotiation (`Accept` header)
- Use `[ApiController]` attribute for automatic model validation and `ProblemDetails` responses

**Follow-up:** What does "stateless" really mean in practice? How do you handle sessions? (You don't — use tokens. Each request contains all info needed.)

**🚩 Red Signal:** Uses `POST` for everything, returns `200 OK` for all responses, or uses action-based URLs like `/api/getUsers`.

---

### 🟢 2. What HTTP status codes should a well-designed API return?

**Hint:**

| Code | Meaning | When to use |
|------|---------|-------------|
| `200 OK` | Success with body | GET, PUT, PATCH responses |
| `201 Created` | Resource created | POST — include `Location` header |
| `204 No Content` | Success, no body | DELETE, PUT when no body returned |
| `400 Bad Request` | Invalid input | Validation errors |
| `401 Unauthorized` | Not authenticated | Missing or invalid token |
| `403 Forbidden` | Authenticated, no permission | Valid token but insufficient rights |
| `404 Not Found` | Resource doesn't exist | GET/PUT/DELETE on missing resource |
| `409 Conflict` | Business conflict | Duplicate creation, concurrency conflict |
| `422 Unprocessable Entity` | Semantic errors | Valid syntax but business rules violated |
| `429 Too Many Requests` | Rate limited | Include `Retry-After` header |
| `500 Internal Server Error` | Unhandled server error | Never expose stack traces |

**Follow-up:** What is `ProblemDetails` (RFC 7807) and how does ASP.NET Core use it? How do you customize error responses?

**🚩 Red Signal:** Returns `200` with an error message in the body, or doesn't distinguish `401` from `403`.

---

### 🟡 3. How do you version a REST API?

**Hint:** Three common approaches:

| Approach | Example | Pros | Cons |
|----------|---------|------|------|
| URL path | `/api/v1/orders` | Simple, visible | Breaks caching between versions |
| Query string | `?api-version=1.0` | Non-intrusive | Easy to forget |
| Header | `Accept: application/vnd.myapi.v1+json` | RESTful purist | Harder to test/discover |

In .NET: use `Asp.Versioning.Http` (formerly `Microsoft.AspNetCore.Mvc.Versioning`).

```csharp
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true; // Response header
});
```

**Follow-up:** How do you deprecate and sunset an API version? What strategy do you use for breaking vs non-breaking changes?

**🚩 Red Signal:** No versioning strategy at all, or breaks existing clients without deprecation period.

---

### 🟡 4. How should you handle pagination, filtering, and sorting?

**Hint:**

```
GET /api/products?page=2&pageSize=20&sort=price:desc&category=electronics&minPrice=10

Response:
{
  "data": [...],
  "pagination": {
    "page": 2,
    "pageSize": 20,
    "totalCount": 1543,
    "totalPages": 78,
    "hasNext": true,
    "hasPrevious": true
  }
}
```

**Cursor-based pagination** (for large datasets): `?after=eyJpZCI6MTAwfQ&limit=20` — more performant, no `OFFSET` penalty. Use when total count is not needed.

**Follow-up:** When should you use cursor-based pagination instead of offset-based? What about filtering with complex operators? (Consider OData or custom filter syntax.)

**🚩 Red Signal:** Returns all records in a single response with no pagination, or implements pagination without total count metadata.

---

### 🟢 5. What is the difference between `PUT` and `PATCH`?

**Hint:** `PUT` — full replacement (idempotent). Send the complete resource. Omitted fields are set to default/null. `PATCH` — partial update. Only send changed fields. Can use JSON Merge Patch (`application/merge-patch+json`) or JSON Patch (`application/json-patch+json`).

```csharp
// PUT — full replacement
[HttpPut("{id}")]
public async Task<IActionResult> Update(int id, ProductDto dto) { ... }

// PATCH — partial update
[HttpPatch("{id}")]
public async Task<IActionResult> Patch(int id,
    JsonPatchDocument<ProductDto> patchDoc) { ... }
```

**Follow-up:** How do you validate a PATCH request? How do you distinguish "set to null" from "field not included"?

**🚩 Red Signal:** Implements `PUT` as a partial update or uses `PATCH` to replace the entire resource.

---

### 🟡 6. What is HATEOAS and is it practical?

**Hint:** Hypermedia As The Engine Of Application State — responses include links to related actions/resources. True REST level 3 (Richardson Maturity Model). Allows clients to discover available actions dynamically.

```json
{
  "id": 42,
  "status": "pending",
  "_links": {
    "self": { "href": "/api/orders/42" },
    "approve": { "href": "/api/orders/42/approve", "method": "POST" },
    "cancel": { "href": "/api/orders/42/cancel", "method": "POST" }
  }
}
```

**Reality:** Most APIs don't implement full HATEOAS. Common pragmatic approach: include `self` links and pagination links only.

**Follow-up:** Have you worked with HATEOAS in production? What was your experience?

**🚩 Red Signal:** Has never heard of HATEOAS, or insists it must be implemented fully for every API.

---

## Minimal APIs & Middleware

### 🟡 7. What are Minimal APIs and when would you use them over controllers?

**Hint:** Introduced in .NET 6. Less ceremony — no controller classes, no `[ApiController]` attribute. Define routes inline with `app.MapGet()`, `app.MapPost()`, etc. Good for: microservices, simple APIs, serverless functions, rapid prototyping.

```csharp
app.MapGet("/api/products/{id}", async (int id, AppDbContext db) =>
    await db.Products.FindAsync(id) is Product p
        ? Results.Ok(p)
        : Results.NotFound());

app.MapPost("/api/products", async (ProductDto dto, AppDbContext db) =>
{
    var product = dto.ToEntity();
    db.Products.Add(product);
    await db.SaveChangesAsync();
    return Results.Created($"/api/products/{product.Id}", product);
});
```

**Follow-up:** How do you organize minimal APIs as the project grows? (Route groups, extension methods, Carter library.) Can you use filters with minimal APIs?

**🚩 Red Signal:** Dismisses minimal APIs without understanding their use case, or puts everything in `Program.cs` for a large application.

---

### 🟡 8. Explain the ASP.NET Core middleware pipeline.

**Hint:** Request flows through middleware in order: each middleware can short-circuit or pass to the next via `next()`. Response flows back in reverse. Order matters: authentication before authorization, CORS before routing, exception handling first.

```csharp
app.UseExceptionHandler("/error");   // 1. Catch all exceptions
app.UseHsts();                        // 2. Security headers
app.UseHttpsRedirection();           // 3. Redirect HTTP to HTTPS
app.UseCors("MyPolicy");             // 4. CORS
app.UseAuthentication();             // 5. Who are you?
app.UseAuthorization();              // 6. Are you allowed?
app.UseRateLimiter();                // 7. Rate limiting
app.MapControllers();                // 8. Route to controllers
```

**Follow-up:** How do you write custom middleware? What's the difference between `Use`, `Map`, and `Run`?

**🚩 Red Signal:** Doesn't understand that middleware order matters or puts authorization before authentication.

---

## Swagger / OpenAPI

### 🟢 9. How do you configure Swagger in ASP.NET Core?

**Hint:** Add `Swashbuckle.AspNetCore` or `NSwag`. In .NET 9+, built-in OpenAPI via `Microsoft.AspNetCore.OpenApi`.

```csharp
// Registration
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new OpenApiInfo { Title = "My API", Version = "v1" });
    c.IncludeXmlComments(xmlPath); // Enable XML doc comments
});

// Pipeline
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}
```

**Best practices:** Use `[ProducesResponseType]`, XML comments, `[SwaggerOperation]`. Disable in production or protect with auth.

**Follow-up:** How do you generate client SDKs from the OpenAPI spec? (NSwag, AutoRest, openapi-generator.)

**🚩 Red Signal:** Ships Swagger UI to production without restrictions, or doesn't document response types and status codes.

---

### 🟡 10. How do you document authentication requirements in Swagger?

**Hint:** Add a security definition and security requirement:

```csharp
c.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
{
    Description = "JWT token in the Authorization header",
    Name = "Authorization",
    In = ParameterLocation.Header,
    Type = SecuritySchemeType.Http,
    Scheme = "bearer",
    BearerFormat = "JWT"
});

c.AddSecurityRequirement(new OpenApiSecurityRequirement
{
    {
        new OpenApiSecurityScheme
        {
            Reference = new OpenApiReference
                { Type = ReferenceType.SecurityScheme, Id = "Bearer" }
        },
        Array.Empty<string>()
    }
});
```

This adds the "Authorize" 🔒 button in Swagger UI.

**Follow-up:** How do you mark some endpoints as public (no auth) in the Swagger doc?

**🚩 Red Signal:** Swagger docs show no auth requirements — consumers don't know the API needs a token.

---

## Machine-to-Machine (M2M) Auth & JWT

### 🟢 11. How does JWT authentication work in ASP.NET Core?

**Hint:** Middleware validates the token signature, issuer (`iss`), audience (`aud`), and expiration (`exp`). Configure with `AddAuthentication().AddJwtBearer()`. Token is passed in the `Authorization: Bearer <token>` header. Claims are extracted and available via `HttpContext.User`.

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = "https://login.microsoftonline.com/{tenantId}/v2.0";
        options.Audience = "api://my-api";
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ClockSkew = TimeSpan.FromMinutes(1)
        };
    });
```

**Follow-up:** What are the three parts of a JWT? How do you inspect a token for debugging? (jwt.io, jwt.ms — never paste production tokens!)

**🚩 Red Signal:** Stores JWTs in local storage for server-rendered apps, or doesn't validate issuer/audience.

---

### 🟡 12. Explain M2M (Machine-to-Machine) authentication with OAuth 2.0 Client Credentials.

**Hint:** No user involved — the client (a service) authenticates with `client_id` + `client_secret` (or certificate) to the token endpoint. Receives an access token scoped to the client (no user claims). Used for service-to-service communication.

```
POST /oauth2/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id=service-a
&client_secret=***
&scope=api://my-api/.default
```

In .NET: use `Microsoft.Identity.Web`, `IdentityModel`, or `HttpClient` with `DelegatingHandler` for automatic token management and caching.

**Follow-up:** How do you secure the client secret? (Key vault, managed identity, certificate-based auth.) What's the difference between client credentials and on-behalf-of flow?

**🚩 Red Signal:** Uses user-based flows (authorization code) for service-to-service, or hard-codes tokens.

---

### 🟡 13. How do you secure an API that is called by both users and other services?

**Hint:** Support multiple authentication schemes. Validate claims to differentiate:
- **User tokens:** have `sub` (subject), `name`, roles, user-specific scopes
- **M2M tokens:** have `azp` or `client_id`, application-level scopes, no `sub`

Apply policies: `[Authorize(Policy = "M2MOnly")]`, `[Authorize(Policy = "UserOnly")]`, `[Authorize(Policy = "Either")]`.

```csharp
builder.Services.AddAuthorizationBuilder()
    .AddPolicy("M2MOnly", p => p.RequireClaim("azp")) // Has app identity
    .AddPolicy("UserOnly", p => p.RequireClaim("sub")); // Has user identity
```

**Follow-up:** How do you implement different authorization logic based on token type in the same endpoint?

**🚩 Red Signal:** Same endpoint logic for user and M2M without checking token type or appropriate scopes.

---

### 🟡 14. What are best practices for token lifetime and refresh?

**Hint:**
- **Access tokens:** Short-lived (5-15 min)
- **Refresh tokens:** Long-lived (hours/days), rotated on use, for user flows only
- **M2M tokens:** No refresh tokens — request a new token when expired. Cache until near expiry
- Never store secrets in client-side code
- Rotate client secrets periodically
- Use certificate credentials for production M2M

**Follow-up:** How do you handle token caching in a multi-instance service? (Distributed cache, `IMemoryCache` with expiry slightly before token expiry.)

**🚩 Red Signal:** Issues access tokens that never expire, or caches M2M tokens without checking expiration.

---

### 🔴 15. How do you handle authorization beyond authentication?

**Hint:**

| Type | Description | Example |
|------|-------------|---------|
| Role-based | Check user role | `[Authorize(Roles = "Admin")]` |
| Claims-based | Check specific claim | `policy.RequireClaim("department", "HR")` |
| Policy-based | Custom logic | `IAuthorizationHandler` implementation |
| Resource-based | Ownership check | "Can this user edit THIS order?" |

```csharp
// Custom authorization handler
public class OrderOwnerHandler :
    AuthorizationHandler<OrderOwnerRequirement, Order>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        OrderOwnerRequirement requirement,
        Order order)
    {
        if (context.User.FindFirst("sub")?.Value == order.UserId)
            context.Succeed(requirement);
        return Task.CompletedTask;
    }
}
```

**Follow-up:** Where should authorization checks live — controllers, services, or middleware? (Centralize in policies/handlers; keep controllers thin.)

**🚩 Red Signal:** All authorization is just `[Authorize]` with no roles/policies, or checks are scattered inconsistently.

---

### 🟡 16. How do you implement rate limiting in ASP.NET Core?

**Hint:** Built-in since .NET 7: `Microsoft.AspNetCore.RateLimiting`. Algorithms: fixed window, sliding window, token bucket, concurrency limiter.

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("api", opt =>
    {
        opt.Window = TimeSpan.FromMinutes(1);
        opt.PermitLimit = 100;
        opt.QueueLimit = 10;
    });
    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
});

app.UseRateLimiter();

// Apply per endpoint
app.MapGet("/api/data", () => "OK").RequireRateLimiting("api");
```

**Follow-up:** How do you apply different rate limits per user/API key? How do rate limits work in a distributed (multi-instance) environment? (Need distributed rate limiting — Redis-based.)

**🚩 Red Signal:** No rate limiting strategy, or implements rate limiting at the application level without considering distributed scenarios.

---

## Scenario Questions

### 💡 17. Design the API for an e-commerce order management system. What endpoints, verbs, and status codes?

**Expected approach:**

```
GET    /api/orders                    → 200 (list with pagination)
GET    /api/orders/{id}               → 200 or 404
POST   /api/orders                    → 201 + Location header
PUT    /api/orders/{id}               → 200 or 404
PATCH  /api/orders/{id}               → 200 or 404
DELETE /api/orders/{id}               → 204 or 404

POST   /api/orders/{id}/cancel        → 200 or 409 (can't cancel shipped)
POST   /api/orders/{id}/ship          → 200 or 409 (already shipped)

GET    /api/orders/{id}/items         → 200 (sub-resource)
POST   /api/orders/{id}/items         → 201
DELETE /api/orders/{id}/items/{itemId} → 204
```

**Discussion points:** Idempotency keys for POST, optimistic concurrency for PUT, validation error format, authentication.

**🚩 Red Signal:** Uses only POST and GET, no proper status codes, no sub-resources.

---

### 💡 18. A third-party service calls your API. How do you authenticate and authorize it?

**Expected approach:**
1. Register the third party as a client in your identity provider (Azure AD, Auth0, etc.)
2. Issue `client_id` + `client_secret` or certificate credentials
3. Third party uses Client Credentials flow to obtain an access token
4. Your API validates the JWT (signature, audience, issuer, expiry)
5. Authorize based on scopes/claims in the token
6. Implement rate limiting per client
7. Log all M2M requests for auditing
8. Rotate secrets on a schedule

**🚩 Red Signal:** Uses static API keys with no rotation, or shares the same credentials across all third parties.
