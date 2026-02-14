# REST API, Swagger & Authentication — Senior Interview Questions

## REST API Conventions

### 1. What are the key REST constraints and how do you apply them in .NET?

**Hint:** Stateless, client-server, cacheable, uniform interface, layered system. In .NET: use HTTP verbs correctly (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`), return proper status codes, use resource-based URIs (nouns not verbs), support content negotiation.

**🚩 Red Signal:** Uses `POST` for everything, returns `200 OK` for all responses, or uses action-based URLs like `/api/getUsers`.

---

### 2. What HTTP status codes should a well-designed API return?

**Hint:** `200 OK` (success), `201 Created` (after POST), `204 No Content` (after DELETE/PUT), `400 Bad Request` (validation), `401 Unauthorized` (no/bad credentials), `403 Forbidden` (authenticated but no permission), `404 Not Found`, `409 Conflict`, `422 Unprocessable Entity`, `500 Internal Server Error`.

**🚩 Red Signal:** Returns `200` with an error message in the body, or doesn't distinguish `401` from `403`.

---

### 3. How do you version a REST API?

**Hint:** URL path (`/api/v1/`), query string (`?api-version=1.0`), header (`Accept: application/vnd.myapi.v1+json`). In .NET use `Asp.Versioning.Http` (formerly `Microsoft.AspNetCore.Mvc.Versioning`). URL versioning is most common and easiest to understand.

**🚩 Red Signal:** No versioning strategy at all, or breaks existing clients without deprecation period.

---

### 4. How should you handle pagination, filtering, and sorting?

**Hint:** Pagination: `?page=1&pageSize=20` or cursor-based with `?after=<token>`. Filtering: `?status=active`. Sorting: `?sort=name:asc`. Return metadata (total count, next/prev links). Avoid returning unbounded datasets.

**🚩 Red Signal:** Returns all records in a single response with no pagination, or implements pagination without total count.

---

### 5. What is the difference between `PUT` and `PATCH`?

**Hint:** `PUT` replaces the entire resource (idempotent). `PATCH` applies a partial update (e.g., JSON Patch or JSON Merge Patch). Omitted fields in `PUT` should be set to default/null; in `PATCH` they are left unchanged.

**🚩 Red Signal:** Implements `PUT` as a partial update or uses `PATCH` to replace the entire resource.

---

## Swagger / OpenAPI

### 6. How do you configure Swagger in ASP.NET Core?

**Hint:** Add `Swashbuckle.AspNetCore` or `NSwag`. Register with `builder.Services.AddSwaggerGen()` and `app.UseSwagger()` / `app.UseSwaggerUI()`. Use XML comments, `[ProducesResponseType]`, and `[SwaggerOperation]` for accurate documentation.

**🚩 Red Signal:** Ships Swagger UI to production without restrictions, or doesn't document response types and status codes.

---

### 7. How do you document authentication requirements in Swagger?

**Hint:** Add a security definition (`AddSecurityDefinition("Bearer", ...)`) and a security requirement (`AddSecurityRequirement(...)`). This adds the "Authorize" button in Swagger UI and documents that endpoints require a JWT.

**🚩 Red Signal:** Swagger docs show no auth requirements — testers/consumers don't know the API needs a token.

---

## Machine-to-Machine (M2M) Auth & JWT

### 8. How does JWT authentication work in ASP.NET Core?

**Hint:** Middleware validates the token signature, issuer, audience, and expiration. Configure with `AddAuthentication().AddJwtBearer()`. Token is passed in the `Authorization: Bearer <token>` header. Claims are extracted and available via `HttpContext.User`.

**🚩 Red Signal:** Stores JWTs in local storage for server-rendered apps, or doesn't validate issuer/audience.

---

### 9. Explain M2M (Machine-to-Machine) authentication with OAuth 2.0 Client Credentials.

**Hint:** No user involved — client authenticates with `client_id` and `client_secret` (or certificate) to the token endpoint. Receives an access token scoped to the client. Used for service-to-service communication. In .NET, use `HttpClient` with token management (e.g., `IdentityModel` or `Microsoft.Identity.Web`).

**🚩 Red Signal:** Uses user-based flows (authorization code) for service-to-service, or hard-codes tokens instead of requesting them from the identity provider.

---

### 10. How do you secure an API that is called by both users and other services?

**Hint:** Support multiple authentication schemes. Validate audience/scope claims to differentiate user tokens from M2M tokens. Apply authorization policies (e.g., `[Authorize(Policy = "M2MOnly")]`). Use scopes for M2M and roles/claims for users.

**🚩 Red Signal:** Same endpoint logic for user and M2M without checking token type or appropriate scopes.

---

### 11. What are best practices for token lifetime and refresh?

**Hint:** Short-lived access tokens (5-15 min). Use refresh tokens for user flows (not applicable for M2M — just request a new token). Cache M2M tokens until near expiry. Never store secrets in client-side code. Rotate client secrets periodically.

**🚩 Red Signal:** Issues access tokens that never expire, or caches M2M tokens without checking expiration.

---

### 12. How do you handle authorization beyond authentication?

**Hint:** Claims-based authorization, policy-based authorization (`RequireClaim`, `RequireRole`, custom `IAuthorizationHandler`). Resource-based authorization for ownership checks. Centralize policies. Keep authorization logic out of controllers — use filters or middleware.

**🚩 Red Signal:** All authorization is just `[Authorize]` with no roles/policies, or authorization checks are scattered inconsistently throughout controller actions.
