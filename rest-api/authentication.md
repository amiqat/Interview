# Authentication & Security

---

### 1. 🟢 What's the difference between 401 and 403?

401 Unauthorized means the client is not authenticated — the server doesn't know who you are (missing or invalid credentials). 403 Forbidden means the client is authenticated but not authorized — the server knows who you are, but you don't have permission to access this resource.

**Hint:** Quick check for understanding of authentication vs authorization at the HTTP level. Follow up: "If a valid user with a 'viewer' role tries to delete a resource, which status code should they get?"

---

### 2. 🟢 Where should a JWT token be sent in an HTTP request?

In the `Authorization: Bearer <token>` header. JWTs should not be sent in query strings (they get logged in server access logs and browser history) or stored in cookies for API-to-API communication (cookies are a browser mechanism and introduce CSRF risks).

**Hint:** Look for awareness of the Bearer token convention and why query strings are insecure for tokens. Follow up: "What's the risk if a JWT ends up in a URL query parameter?"

---

### 3. 🟡 A security reviewer looks at your auth system and asks: "JWTs are just Base64-encoded — anyone can decode the payload and read it. How is this secure?" Walk them through the JWT structure and explain why this isn't the vulnerability they think it is.

A JWT has three Base64URL-encoded parts separated by dots: Header, Payload, and Signature. The header specifies the algorithm (e.g., `HS256` or `RS256`), the payload contains claims like `sub`, `iss`, `exp`, and custom data, and the signature is a cryptographic hash of the first two parts using a secret or private key.

You're right that anyone can decode and read the payload — JWTs are not encrypted by default, so they provide integrity, not confidentiality. The signature ensures that if anyone tampers with the payload (changing a role from "user" to "admin," for example), the signature won't match and the server will reject the token.

Validation involves verifying the signature, then checking that `exp` (expiration) hasn't passed, `nbf` (not before) is in the past, and `iss` (issuer) and `aud` (audience) match expected values.

With symmetric signing (HMAC), both parties share the same secret; with asymmetric signing (RSA/ECDSA), the issuer signs with a private key and consumers verify with the public key — this is why asymmetric signing is preferred for distributed systems. Sensitive data like passwords should never be placed in the payload; if payload confidentiality is needed, use JWE (JSON Web Encryption).

**Hint:** Look for a clear explanation that JWTs provide integrity (tamper detection) not confidentiality, and that the signature is what makes them secure. A strong candidate explains symmetric vs asymmetric signing and knows that sensitive data shouldn't go in the payload. Follow up: "What happens if someone steals a valid JWT — how do you revoke it?"

**🚩 Red Signal:** Believes JWT payloads are encrypted, stores passwords or secrets in the payload, or doesn't validate claims like `exp` and `aud` on the server.

---

### 4. 🟡 Two backend services in your system need to communicate securely — there's no user involved and no browser. How do you set up the authentication between them?

You use the OAuth 2.0 Client Credentials flow, which is designed for machine-to-machine (M2M) communication. The calling service authenticates with the authorization server by sending its `client_id` and `client_secret` to the token endpoint with `grant_type=client_credentials`. The authorization server validates the credentials and returns a JWT access token, which the calling service attaches to requests as a `Bearer` token.

The receiving service validates the JWT signature, issuer, audience, and expiration — it never sees or needs the client secret. Tokens should be cached until they are close to expiration to avoid requesting a new token on every call.

The client secret must be stored securely (environment variable, Azure Key Vault, AWS Secrets Manager), transmitted only over HTTPS, and rotated periodically. For higher security, use `private_key_jwt` authentication where the client signs an assertion with its private key instead of sending a shared secret over the wire.

```
POST /oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id=service-a
&client_secret=<secret>
&scope=orders.read
```

**Hint:** Look for understanding of the Client Credentials grant and how it differs from user-facing flows (no redirect, no user context). A strong candidate discusses token caching, secret rotation, and the `private_key_jwt` alternative. Follow up: "How do you scope down what service A can do on service B — can you give it just read access?"

**🚩 Red Signal:** Sends client secrets in query strings, logs tokens or secrets, or uses a user-facing OAuth flow for service-to-service communication.

---

### 5. 🟡 You're adding JWT authentication to an ASP.NET Core API. Some endpoints should be protected, but the health check and login endpoints must stay public. Walk through how you set this up.

You register JWT Bearer authentication in the DI container and configure `TokenValidationParameters` to tell the framework how to validate incoming tokens — which issuer and audience to accept, and which signing key to use.

The middleware pipeline order is critical: `UseAuthentication()` must come before `UseAuthorization()`. Protect endpoints by default using `[Authorize]` at the controller level or as a global filter, then opt out specific endpoints with `[AllowAnonymous]` for health checks and login.

For more granular control, define authorization policies that check for specific claims or roles — for example, requiring an "admin" role for management endpoints. The `AddJwtBearer` configuration reads signing keys, which for asymmetric setups can be fetched automatically from the identity provider's JWKS endpoint.

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = "https://auth.example.com";
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidIssuer = "https://auth.example.com",
            ValidateAudience = true,
            ValidAudience = "orders-api",
            ValidateLifetime = true
        };
    });

builder.Services.AddAuthorization();

app.UseAuthentication();
app.UseAuthorization();
```

```csharp
[Authorize]
[ApiController]
[Route("orders")]
public class OrdersController : ControllerBase { /* protected */ }

[AllowAnonymous]
[ApiController]
[Route("health")]
public class HealthController : ControllerBase { /* public */ }
```

**Hint:** Look for correct middleware ordering and understanding of `TokenValidationParameters`. A strong candidate explains the difference between `[Authorize]` and `[AllowAnonymous]`, mentions policy-based authorization for role or claim checks, and can describe how keys are discovered via JWKS. Follow up: "What happens if you swap the order of `UseAuthentication` and `UseAuthorization`?"

**🚩 Red Signal:** Places `UseAuthorization()` before `UseAuthentication()`, doesn't validate issuer or audience, or hard-codes signing keys in source code.

---

### 6. 🟢 Your API returns 401 Unauthorized when a valid, authenticated user tries to access an admin-only endpoint. A junior developer says "the authentication is broken." Is that the right diagnosis?

No — the authentication is working correctly; it's the authorization that is rejecting the request. Authentication answers "who are you?" and authorization answers "are you allowed to do this?"

The user successfully authenticated (the server knows who they are via a valid JWT), but their identity lacks the required claims or roles to access the admin endpoint. The correct response is actually 403 Forbidden, not 401 Unauthorized. A 401 means "I don't know who you are — provide credentials," while 403 means "I know who you are, but you can't access this."

In ASP.NET Core, authentication is handled by `UseAuthentication()` middleware and the `[Authorize]` attribute, while fine-grained authorization uses policies, roles, and claims — for example, `[Authorize(Roles = "Admin")]` or `[Authorize(Policy = "RequireAdminClaim")]`. However, ASP.NET Core's default behavior returns 401 for both failed authentication and failed authorization when using JWT Bearer, which can cause exactly this confusion — configuring `ForbiddenStatusCode` or handling the `OnForbidden` event can correct this.

**Hint:** Look for a clear distinction between authentication (identity) and authorization (permissions), and correct mapping to HTTP status codes. A strong candidate knows that ASP.NET Core can conflate 401/403 in some configurations and explains claims-based vs role-based vs policy-based authorization. Follow up: "How would you implement resource-based authorization — for example, users can only edit their own orders?"

**🚩 Red Signal:** Conflates authentication with authorization, or doesn't know the difference between 401 and 403.

---

### 7. 🟢 Your team builds a SPA hosted at `app.example.com` that calls an API at `api.example.com`. Requests work perfectly in Postman, but the browser blocks them with a CORS error. The junior developer suggests disabling CORS. Why does this happen, and what's the proper fix?

Browsers enforce the Same-Origin Policy, which blocks JavaScript from making requests to a different origin (scheme + host + port). Postman doesn't enforce this because it's not a browser — it sends requests directly without the preflight check.

CORS (Cross-Origin Resource Sharing) is the mechanism that lets the server explicitly whitelist which origins are allowed. When the browser sees a cross-origin request, it sends a preflight `OPTIONS` request with `Origin`, `Access-Control-Request-Method`, and `Access-Control-Request-Headers` headers; the server must respond with the appropriate `Access-Control-Allow-*` headers.

In ASP.NET Core, you configure CORS by registering a policy with `AddCors` and applying it with `UseCors`. The fix is to allow `app.example.com` specifically — never use `AllowAnyOrigin` combined with `AllowCredentials`, as the spec forbids this combination for security reasons.

```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowSpa", policy =>
    {
        policy.WithOrigins("https://app.example.com")
              .AllowAnyHeader()
              .AllowAnyMethod()
              .AllowCredentials();
    });
});

app.UseCors("AllowSpa");
```

**Hint:** Look for understanding of why browsers enforce CORS but tools like Postman don't, and the difference between simple requests and preflight requests. A strong candidate explains that `AllowAnyOrigin` with `AllowCredentials` is invalid and knows where `UseCors` must be placed in the middleware pipeline. Follow up: "What's the difference between a simple CORS request and one that triggers a preflight?"

**🚩 Red Signal:** Suggests disabling CORS entirely in production, uses `AllowAnyOrigin` with `AllowCredentials`, or doesn't understand why Postman behaves differently from the browser.
