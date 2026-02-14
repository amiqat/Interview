# API Design

---

### 1. 🟢 You're reviewing a pull request for a new API. The endpoints are named `/getOrders`, `/createOrder`, and `/deleteOrderById`. Every response returns HTTP 200, even when validation fails — the error is embedded in the JSON body. What feedback do you give in the review?

The endpoint names violate REST conventions by embedding verbs — resources should be nouns. The correct design uses `/orders` as the resource with HTTP methods conveying the action: `GET /orders` to list, `POST /orders` to create, `DELETE /orders/{id}` to remove.

Returning 200 for everything defeats the purpose of HTTP status codes and breaks client error-handling logic. A validation failure should return 400 Bad Request, a missing resource 404, and a successful creation 201 Created. Consistent use of status codes lets clients, proxies, and monitoring tools behave correctly without parsing the body.

Additionally, plural nouns should be used consistently (`/orders`, not `/order`), and naming should follow one convention (kebab-case or camelCase) throughout the API. A well-designed API also supports pagination, filtering, and sorting via query parameters for collection endpoints.

**Hint:** Look for understanding of resource-oriented design and correct HTTP method semantics. A strong candidate discusses idempotency of PUT and DELETE, the difference between 401 and 403, and mentions HATEOAS as an advanced concept. Follow up: "What status code would you return for a duplicate order — 400 or 409?"

**🚩 Red Signal:** Uses GET for creating or mutating resources, believes 200 is the only success code needed, or puts verbs in every URL.

---

### 2. 🟡 Your public API has been live for a year with paying customers. Product asks you to make breaking changes to the order resource — renaming fields and removing deprecated ones. How do you roll this out without breaking existing clients?

You introduce API versioning so the old contract remains available while new clients adopt the updated one. The most common approach in ASP.NET Core is URL-path versioning (`/v1/orders` and `/v2/orders`) because it is explicit and easy to test in a browser or with curl.

Alternatives include query-string versioning (`?api-version=2`), custom header versioning (`X-Api-Version: 2`), and media-type versioning (`Accept: application/vnd.myapi.v2+json`) — each has trade-offs between REST purity and developer ergonomics. In .NET, the `Asp.Versioning.Mvc` package supports all of these strategies and lets you mark old versions as deprecated while keeping them functional.

You should communicate a deprecation timeline, return `Sunset` or custom deprecation headers, and monitor traffic on the old version before removing it. Running both versions simultaneously means maintaining two controller sets or using version-aware mapping, so the migration window should be kept as short as practical.

| Strategy | Pros | Cons |
|---|---|---|
| URL path (`/v1/orders`) | Simple, explicit, easy to test | Breaks URI purity |
| Query string (`?api-version=1`) | Non-intrusive | Easy to forget, caching issues |
| Header (`Api-Version: 1`) | Cleaner REST semantics | Harder to test in browser |
| Media type versioning | Full content negotiation | Complex to implement and document |

**Hint:** Look for a concrete versioning strategy and awareness of the operational side — deprecation headers, monitoring old-version traffic, documentation updates. Follow up: "How would you handle a client that never migrates off v1?"

**🚩 Red Signal:** Ships a public API with no versioning strategy, or suggests just changing the existing endpoints and hoping clients adapt.

---

### 3. 🟢 A new team member asks how external developers will know what your API expects and returns. You want documentation that is auto-generated and always matches the actual code. How do you set this up?

You integrate OpenAPI (formerly Swagger) into the ASP.NET Core project. Adding `Swashbuckle.AspNetCore` or `NSwag` generates a machine-readable `openapi.json` spec directly from your controllers, models, and route metadata. Enrich the spec with `[ProducesResponseType(typeof(OrderDto), 200)]` attributes and XML documentation comments — enable XML doc generation in the `.csproj` and configure Swashbuckle to read it. The Swagger UI provides an interactive explorer where developers can try endpoints directly. Beyond documentation, the OpenAPI spec can generate typed client SDKs in any language using tools like NSwag, AutoRest, or openapi-generator, which eliminates manual HTTP client code. In production, consider restricting Swagger UI access behind authentication or disabling it entirely, while still publishing the spec file for tooling.

```csharp
// Program.cs
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new() { Title = "Orders API", Version = "v1" });
    var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
    c.IncludeXmlComments(Path.Combine(AppContext.BaseDirectory, xmlFile));
});

app.UseSwagger();
app.UseSwaggerUI();
```

**Hint:** Look for awareness that OpenAPI is a spec, not just a UI — the real value is the machine-readable contract. A strong candidate mentions client generation, operation filters for customizing the spec, and protecting the Swagger UI in production. Follow up: "Have you ever generated a client from an OpenAPI spec? What was the experience like?"

**🚩 Red Signal:** Has never looked at the generated `openapi.json`, or confuses Swagger UI with the OpenAPI specification itself.

---

### 4. 🟡 Your API returns plain strings for some errors, HTML error pages for others (from the framework), and JSON for yet others. A frontend developer complains they can't reliably parse error responses. How do you standardize this?

You adopt the Problem Details standard (RFC 9457, formerly RFC 7807), which defines a consistent JSON structure for HTTP API errors with fields like `type`, `title`, `status`, `detail`, and `instance`. In .NET 7+, calling `builder.Services.AddProblemDetails()` configures the framework to produce Problem Details responses for exceptions, model validation failures, and empty-result status codes automatically. For validation errors, ASP.NET Core returns `ValidationProblemDetails`, which extends Problem Details with an `errors` dictionary keyed by field name. You can customize the output using `IExceptionHandler` in .NET 8+ for global exception handling or by configuring `ProblemDetailsOptions` to add custom fields. This ensures that every error — whether a 400, 404, 500, or anything else — follows the same envelope, making client-side error handling straightforward and consistent.

```json
{
  "type": "https://tools.ietf.org/html/rfc9110#section-15.5.5",
  "title": "Not Found",
  "status": 404,
  "detail": "Order with ID 42 was not found.",
  "instance": "/orders/42"
}
```

**Hint:** Look for familiarity with the Problem Details RFC and awareness that ASP.NET Core has built-in support. A strong candidate mentions `IExceptionHandler` for centralized error processing and can explain the difference between `ProblemDetails` and `ValidationProblemDetails`. Follow up: "How would you add custom fields to the problem details response — for example, a correlation ID?"

**🚩 Red Signal:** Returns inconsistent error shapes depending on the endpoint, or catches exceptions and manually writes arbitrary JSON error objects everywhere.

---

### 5. 🟡 Your API is being hammered by a bot doing 10,000 requests per minute, and some of your endpoints are expensive (large database joins, external service calls). How do you protect the API without blocking legitimate users?

You implement rate limiting using the built-in rate-limiting middleware available in .NET 7+. The middleware supports multiple algorithms: fixed window (simple counter per time window), sliding window (smoother traffic shaping), token bucket (allows short bursts), and concurrency limiter (caps simultaneous requests). You configure policies and apply them globally, per-endpoint, or per-controller. Rate limits should be keyed by client identifier — IP address, API key, or authenticated user — so one abusive client doesn't consume the quota for everyone. When a client exceeds the limit, the API returns `429 Too Many Requests` with a `Retry-After` header. For distributed deployments behind a load balancer, you need a shared store like Redis so the counter is consistent across instances.

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("api", opt =>
    {
        opt.PermitLimit = 100;
        opt.Window = TimeSpan.FromMinutes(1);
        opt.QueueLimit = 0;
    });
    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
});

app.UseRateLimiter();
```

**Hint:** Look for knowledge of different rate-limiting algorithms and when each is appropriate. A strong candidate discusses per-client keying, distributed rate limiting with Redis, and mentions that expensive endpoints might need stricter limits than cheap ones. Follow up: "How would you rate-limit differently for authenticated vs anonymous users?"

**🚩 Red Signal:** Has no strategy for protecting APIs from abuse, or suggests only client-side throttling as the solution.
