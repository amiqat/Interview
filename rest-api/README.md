# REST API, Swagger & Authentication

Questions on RESTful API conventions, OpenAPI/Swagger documentation, machine-to-machine authentication, and JWT.

---

### 1. 🟢 What are the key conventions of a well-designed RESTful API?

- **Nouns for resources**, not verbs: `/orders`, not `/getOrders`.
- **HTTP methods map to operations**: GET (read), POST (create), PUT (full update), PATCH (partial update), DELETE.
- **Proper status codes**: 200 OK, 201 Created, 204 No Content, 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 409 Conflict, 422 Unprocessable Entity.
- **Consistent naming**: plural nouns, kebab-case or camelCase (pick one).
- **Pagination, filtering, sorting** via query parameters.
- **Versioning** via URL path (`/v1/`) or header (`Accept: application/vnd.api+json;version=1`).

**Hint:** A senior candidate should also discuss HATEOAS (even if not always implemented), idempotency of PUT/DELETE, and the difference between 401 and 403.

**🚩 Red Signal:** Uses GET for creating resources, returns 200 for everything, or puts verbs in URLs.

---

### 2. 🟡 How do you version a REST API and what are the trade-offs of each approach?

| Strategy | Pros | Cons |
|---|---|---|
| URL path (`/v1/orders`) | Simple, explicit, easy to route | Breaks URI purity, hard to sunset |
| Query string (`?api-version=1`) | Non-intrusive | Easy to forget, less discoverable |
| Header (`Accept: ...;v=1`) | Cleanest REST, same URI | Harder to test in browser |
| Media type versioning | Full content negotiation | Complex to implement |

**Hint:** In ASP.NET Core, the `Asp.Versioning.Mvc` package supports all strategies. The candidate should express a preference and justify it.

**🚩 Red Signal:** Ships a public API with no versioning strategy at all.

---

### 3. 🟢 How do you integrate Swagger/OpenAPI with an ASP.NET Core API?

Add `Swashbuckle.AspNetCore` or `NSwag` and call `builder.Services.AddSwaggerGen()` + `app.UseSwagger()` + `app.UseSwaggerUI()`. Annotations like `[ProducesResponseType]`, XML comments, and `[SwaggerOperation]` enrich the spec.

**Hint:** A great answer covers: generating strongly-typed clients from the spec (NSwag, AutoRest, openapi-generator), customising the spec with operation filters, and securing the Swagger UI endpoint in production.

**🚩 Red Signal:** Has never looked at the generated OpenAPI JSON or does not know that Swagger is just a UI for the OpenAPI specification.

---

### 4. 🟡 Explain the structure of a JWT. How does the server validate it?

A JWT has three Base64URL-encoded parts separated by dots: **Header** (algorithm, type), **Payload** (claims: `sub`, `iss`, `aud`, `exp`, etc.), and **Signature** (HMAC or RSA/ECDSA over header + payload).

Validation steps:
1. Decode the header to determine the signing algorithm.
2. Verify the signature using the secret or public key.
3. Check standard claims: `exp` (not expired), `nbf` (not before), `iss` (issuer), `aud` (audience).
4. Check custom claims for authorization.

**Hint:** The candidate should explain the difference between symmetric (HMAC-SHA256) and asymmetric (RSA, ECDSA) signing and when each is appropriate (asymmetric for distributed systems where the verifier should not have the signing key).

**🚩 Red Signal:** Stores sensitive data (passwords, PII) in the JWT payload without encryption, or does not validate `exp`/`iss`/`aud`.

---

### 5. 🟡 How do you implement machine-to-machine (M2M) authentication using OAuth 2.0 Client Credentials flow?

The client sends its `client_id` and `client_secret` to the token endpoint (`/connect/token`) with `grant_type=client_credentials` and the requested `scope`. The authorization server validates the credentials and returns an access token (JWT). The client includes this token in the `Authorization: Bearer <token>` header of subsequent API requests.

**Hint:** The candidate should mention token caching (tokens have an `expires_in`), using HTTPS for all token exchanges, rotating client secrets, and the option of using client certificate authentication (`private_key_jwt`) instead of a shared secret.

**🚩 Red Signal:** Sends client secrets in query strings or logs them, or does not understand the difference between client credentials flow and authorization code flow.

---

### 6. 🟡 How do you secure a REST API endpoint in ASP.NET Core using JWT Bearer authentication?

1. Register the authentication scheme: `builder.Services.AddAuthentication().AddJwtBearer(options => { ... })`.
2. Configure `TokenValidationParameters`: `ValidIssuer`, `ValidAudience`, `IssuerSigningKey`, `ValidateLifetime`.
3. Add `app.UseAuthentication()` and `app.UseAuthorization()` middleware in the correct order.
4. Protect endpoints with `[Authorize]` attribute or policy-based authorization.

**Hint:** Look for understanding of the middleware pipeline order (Authentication before Authorization), custom policies (`RequireClaim`, `RequireRole`), and the difference between `[Authorize]` and `[AllowAnonymous]`.

**🚩 Red Signal:** Places `UseAuthorization` before `UseAuthentication`, or hard-codes tokens in source code.

---

### 7. 🟢 What is the difference between authentication and authorization in the context of REST APIs?

**Authentication** verifies *who* the caller is (identity). **Authorization** determines *what* the caller is allowed to do (permissions). In ASP.NET Core, authentication populates `HttpContext.User` (ClaimsPrincipal), and authorization evaluates policies against those claims.

**Hint:** A senior candidate should discuss claims-based identity, role-based vs policy-based vs resource-based authorization, and how `IAuthorizationHandler` enables custom logic.

**🚩 Red Signal:** Conflates authentication with authorization or implements authorization checks by parsing the JWT manually in every controller action.

---

### 8. 🟡 How do you handle API error responses consistently?

Use the **Problem Details** standard (RFC 7807 / RFC 9457): return a JSON body with `type`, `title`, `status`, `detail`, and optional `extensions`. In ASP.NET Core 7+, call `builder.Services.AddProblemDetails()` to standardise error responses across exceptions, model validation errors, and status codes.

**Hint:** A strong answer includes global exception handling middleware, `IExceptionHandler` (ASP.NET Core 8+), and how `ValidationProblemDetails` differs from `ProblemDetails`.

**🚩 Red Signal:** Returns plain strings or inconsistent error shapes (sometimes JSON, sometimes HTML, sometimes stack traces in production).

---

### 9. 🟡 How do you implement rate limiting in ASP.NET Core?

ASP.NET Core 7+ provides built-in rate limiting middleware: `builder.Services.AddRateLimiter(options => { ... })` with policies like fixed window, sliding window, token bucket, and concurrency limiter. Apply per-endpoint with `[EnableRateLimiting("policy")]` or globally.

**Hint:** Discuss the difference between client-side and server-side rate limiting, using `X-RateLimit-*` headers to communicate limits, and distributed rate limiting with Redis for multi-instance deployments.

**🚩 Red Signal:** Has no strategy for protecting APIs from abuse or relies solely on an external gateway without understanding the mechanisms.

---

### 10. 🟢 What is CORS and how do you configure it in ASP.NET Core?

Cross-Origin Resource Sharing (CORS) allows browsers to make requests to a different origin. Configure with `builder.Services.AddCors(options => { options.AddPolicy("name", policy => policy.WithOrigins(...).AllowAnyMethod().AllowAnyHeader()); })` and `app.UseCors("name")`.

**Hint:** The candidate should know the difference between simple and preflight requests, why `AllowAnyOrigin` combined with `AllowCredentials` is forbidden by the spec, and that CORS is a browser-only protection — it does not secure the API from non-browser clients.

**🚩 Red Signal:** Disables CORS entirely with `AllowAnyOrigin().AllowAnyMethod().AllowAnyHeader()` in production without understanding the implications.
