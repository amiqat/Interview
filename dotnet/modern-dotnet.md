# Modern .NET — Minimal APIs, .NET 8/10, DI & Hosting

---

### 1. 🟢 What HTTP status code means "Created"? What about "Too Many Requests"?

**201 Created** — returned after a successful POST that creates a resource. The response should include a `Location` header pointing to the new resource. **429 Too Many Requests** — the client has exceeded a rate limit. The response should include a `Retry-After` header indicating when the client can retry.

**Hint:** Follow-up: "What's the difference between 401 and 403?" (401 = not authenticated, 403 = authenticated but not authorized.) "When would you return 202 instead of 201?" (When the creation is asynchronous — accepted for processing but not yet completed.)

**🚩 Red Signal:** Confuses 401/403, or doesn't know common status codes beyond 200/404/500.

---

### 2. 🟢 What are the three DI lifetimes in ASP.NET Core and when do you use each?

**Singleton** — one instance for the entire application lifetime. Use for stateless services, caches, and configuration holders. **Scoped** — one instance per request (per `IServiceScope`). Use for EF Core `DbContext`, unit-of-work patterns, and per-request state. **Transient** — a new instance every time it's requested. Use for lightweight, stateless services where sharing state across injections would be a bug.

**Hint:** Follow-up: "What happens if you inject a Scoped service into a Singleton?" (Captive dependency — the Scoped instance is captured and lives as long as the Singleton, causing stale data and potential thread-safety issues. .NET throws at startup if `ValidateScopes` is enabled.) A strong answer mentions `ValidateScopes` and `ValidateOnBuild`.

**🚩 Red Signal:** Cannot name all three lifetimes, or doesn't understand the captive dependency problem.

---

### 3. 🟢 Name two things Native AOT cannot do.

Native AOT compiles IL to native code ahead of time, so it **cannot use `Reflection.Emit`** (no JIT to compile dynamically generated IL) and **cannot load assemblies dynamically** at runtime (`Assembly.LoadFile` / `Assembly.Load` from bytes). This means libraries relying on runtime code generation (e.g., some serializers, ORMs with proxy generation, Castle DynamicProxy) won't work without source generators or compile-time alternatives.

**Hint:** Follow-up: "How does System.Text.Json work with Native AOT?" (Source generators — `JsonSerializerContext` generates serialization code at compile time.) "What about EF Core?" (Limited support — query compilation uses expressions, not `Reflection.Emit`, but some features are restricted.)

**🚩 Red Signal:** Doesn't know what Native AOT is, or believes it's just a faster JIT.

---

### 4. 🟡 When would you choose minimal APIs over controllers? What are the trade-offs?

**Minimal APIs** are ideal for microservices, lightweight HTTP APIs, and projects where you want a thin layer with minimal ceremony — define routes inline in `Program.cs` with lambda handlers. They're faster to bootstrap, have slightly better performance (less middleware overhead), and align well with vertical-slice architecture. **Controllers** are better for large APIs with complex routing, model validation, content negotiation, and when you want the conventions MVC provides (filters, model binding, `[ApiController]` behaviors).

Trade-offs: minimal APIs lack built-in support for some controller features — no `[FromBody]` model validation via `[ApiController]`, no action filters (though endpoint filters fill this gap in .NET 7+). They can become unwieldy when you have dozens of endpoints in one file. Controllers provide a more structured, discoverable organization by default. The performance difference is marginal for most apps — choose based on project complexity and team familiarity.

**Hint:** Look for nuance — not "minimal APIs are always better" or "controllers are always better." A strong answer mentions endpoint filters as the minimal API equivalent of action filters. Follow-up: "How do you organize a minimal API project with 50+ endpoints?" (Route groups, extension methods, Carter library, or vertical-slice folders.)

**🚩 Red Signal:** Believes minimal APIs are a toy/demo feature not suitable for production, or cannot name a single trade-off.

---

### 5. 🟡 What is the middleware pipeline order in ASP.NET Core and why does it matter?

Middleware executes in the **order it's registered** in `Program.cs` (or `Startup.Configure`), forming a pipeline where each middleware can short-circuit or pass to the next. The order matters because each middleware only sees what the previous ones have set up. A typical correct order:

1. `UseExceptionHandler` / `UseDeveloperExceptionPage` — catches exceptions from everything below
2. `UseHttpsRedirection`
3. `UseStaticFiles` — serves files before routing runs
4. `UseRouting` — matches the endpoint
5. `UseCors` — must be between routing and authorization
6. `UseAuthentication` — sets `HttpContext.User`
7. `UseAuthorization` — checks policies against the authenticated user
8. `MapControllers` / `MapEndpoints` — terminal middleware

If you put `UseAuthorization` before `UseAuthentication`, authorization runs against an anonymous user and every request fails. If `UseExceptionHandler` is registered after your middleware, exceptions in that middleware won't be caught.

**Hint:** Ask: "What happens if you put `UseCors` after `UseAuthorization`?" (CORS preflight requests get rejected by auth before CORS middleware can handle them.) A strong answer knows that `UseRouting` and endpoint execution are separate steps, and that middleware between them can inspect the matched endpoint metadata.

**🚩 Red Signal:** Doesn't know that order matters, or has never thought about the pipeline beyond "it just works."

---

### 6. 🟡 What changed from `Startup.cs` to the new `WebApplicationBuilder` pattern, and why?

The old pattern used `Startup.cs` with two convention methods: `ConfigureServices(IServiceCollection)` for DI registration and `Configure(IApplicationBuilder)` for middleware. `WebApplicationBuilder` (introduced in .NET 6) merges both into `Program.cs` — services go on `builder.Services`, middleware goes on `app` after `builder.Build()`. This eliminates the implicit convention-based discovery of `Startup` methods and makes the bootstrap flow explicit and top-to-bottom.

Under the hood, `WebApplicationBuilder` combines `HostBuilder` and the web-specific configuration that previously required `IWebHostBuilder`. It also enables top-level statements (no `Main` method boilerplate), supports `args` for configuration, and auto-adds common defaults (Kestrel, logging, configuration from `appsettings.json`). The old `Startup.cs` pattern still works via `builder.Host.ConfigureWebHostDefaults`, but new projects should use the builder pattern for simplicity.

**Hint:** Follow-up: "How do you organize a large `Program.cs` that's growing unwieldy?" (Extract service registration and middleware into extension methods — e.g., `builder.Services.AddApplicationServices()`, `app.UseApplicationMiddleware()`.) "Can you still use `Startup.cs`?" (Yes — `builder.Host.ConfigureWebHostDefaults(wb => wb.UseStartup<Startup>())`.)

**🚩 Red Signal:** Cannot explain the difference, or believes the two patterns are completely unrelated rather than an evolution.

---

### 7. 🟡 Your minimal API project has grown to 50+ endpoints, all defined in `Program.cs`. It's hard to navigate and developers keep getting merge conflicts. How do you organize it?

Use **route groups** (`app.MapGroup("/api/orders")`) combined with **extension methods** to move endpoint definitions into separate files organized by feature or resource. Each file defines a static method like `MapOrderEndpoints(this RouteGroupBuilder group)` that registers its routes on the group. This keeps `Program.cs` clean — it just calls `app.MapGroup("/api/orders").MapOrderEndpoints()`.

```csharp
// Program.cs — clean and scannable
app.MapGroup("/api/orders").MapOrderEndpoints();
app.MapGroup("/api/products").MapProductEndpoints();

// OrderEndpoints.cs
public static class OrderEndpoints
{
    public static RouteGroupBuilder MapOrderEndpoints(this RouteGroupBuilder group)
    {
        group.MapGet("/", GetOrders);
        group.MapPost("/", CreateOrder);
        group.MapGet("/{id}", GetOrderById);
        return group;
    }
}
```

Route groups also let you apply shared filters, tags, and metadata (e.g., `.RequireAuthorization()`) to all endpoints in the group at once. For larger projects, libraries like **Carter** provide a more structured convention. The key principle is the same as with controllers: one file per feature/resource, discovered and registered centrally.

**Hint:** Look for awareness of `MapGroup` (introduced in .NET 7) and the extension method pattern. Follow-up: "How do you apply authorization to a whole group?" (`.RequireAuthorization()` on the `RouteGroupBuilder`.) "What about endpoint filters?" (`.AddEndpointFilter<T>()` on the group applies to all child endpoints.)

**🚩 Red Signal:** Suggests keeping everything in `Program.cs` and "just using regions," or is unaware of `MapGroup`.

---

### 8. 🟡 A developer injected a Scoped `DbContext` into a Singleton background service. Users report stale data — reads return old values even after writes. What happened?

This is a **captive dependency**. The Singleton service is created once and lives for the entire application lifetime. When the Scoped `DbContext` is injected into it, that `DbContext` instance is also captured and held forever — it never gets disposed or recreated per request. The `DbContext`'s change tracker caches entity states from the first queries, so subsequent reads return tracked (stale) entities instead of querying the database.

The fix: never inject Scoped services directly into Singletons. Instead, inject `IServiceScopeFactory`, create a scope per operation, and resolve the `DbContext` from that scope.

```csharp
public class OrderProcessor : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;

    public OrderProcessor(IServiceScopeFactory scopeFactory) => _scopeFactory = scopeFactory;

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            using var scope = _scopeFactory.CreateScope();
            var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
            // fresh DbContext per iteration — no stale tracking
            await ProcessOrdersAsync(db, ct);
        }
    }
}
```

**Hint:** .NET will throw at startup if `ValidateScopes` is enabled (it's on by default in Development). Follow-up: "Why doesn't this throw in Production?" (`ValidateScopes` defaults to `false` in non-Development environments for performance.) "What about injecting `IDbContextFactory<T>` instead?" (Also valid — creates a new `DbContext` per call without manual scope management.)

**🚩 Red Signal:** Doesn't understand DI lifetimes, or suggests making the `DbContext` a Singleton to "fix" the problem.

---

### 9. 🟡 Kubernetes keeps restarting your pods even though the application runs fine. There are no health checks configured. How do you fix this?

Without health check endpoints, Kubernetes has no way to know if your app is healthy — it falls back to checking if the process is running, or if liveness/readiness probes fail (returning non-200), it restarts the pod. If you've defined probes in your deployment YAML pointing to paths that don't exist, every probe returns 404 and Kubernetes interprets that as unhealthy.

**Fix:** Add ASP.NET Core health checks and map them to the paths your Kubernetes probes expect:

```csharp
builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy())          // basic liveness
    .AddNpgSql(connectionString)                                   // readiness — DB reachable?
    .AddCheck<CustomStartupCheck>("startup");                      // startup probe

app.MapHealthChecks("/healthz/live", new() { Predicate = r => r.Name == "self" });
app.MapHealthChecks("/healthz/ready", new() { Predicate = r => r.Name != "self" && r.Name != "startup" });
app.MapHealthChecks("/healthz/startup", new() { Predicate = r => r.Name == "startup" });
```

Then configure the Kubernetes deployment with matching probes — `livenessProbe` on `/healthz/live`, `readinessProbe` on `/healthz/ready`, and `startupProbe` on `/healthz/startup`. The startup probe prevents restarts during slow initialization (e.g., migrations, cache warmup). The readiness probe removes the pod from the Service if dependencies (DB, cache) are down, without killing it.

**Hint:** Look for understanding of the difference between liveness (is the process hung?), readiness (can it serve traffic?), and startup (is it still initializing?). Follow-up: "What happens if your readiness check is too aggressive — e.g., it fails when a non-critical dependency is down?" (The pod gets pulled from the load balancer unnecessarily — degrade gracefully instead.)

**🚩 Red Signal:** Doesn't know what liveness/readiness probes are, or suggests disabling Kubernetes probes entirely.

---

### 10. 🔴 You're deploying a .NET API to a serverless platform with a strict 200 ms cold-start requirement. JIT warmup adds ~2 seconds on first request. How do you eliminate the JIT overhead?

The primary solution is **Native AOT** (`<PublishAot>true</PublishAot>`) — it compiles IL to native machine code at publish time, producing a self-contained executable with no JIT needed at runtime. Cold starts drop from seconds to low milliseconds because there's no CLR startup, no JIT compilation, and a much smaller memory footprint. The resulting binary is a single native executable with no .NET runtime dependency.

```xml
<!-- .csproj -->
<PropertyGroup>
    <PublishAot>true</PublishAot>
    <InvariantGlobalization>true</InvariantGlobalization>
</PropertyGroup>
```

The trade-offs are significant: no `Reflection.Emit`, no dynamic assembly loading, and libraries must support AOT (via source generators or trimming annotations). Serialization must use source-generated `JsonSerializerContext`. Some EF Core features are limited. You need thorough trimming analysis (`<PublishTrimmed>true</PublishTrimmed>` with `IsTrimmable`) to ensure nothing critical is trimmed away.

If Native AOT isn't feasible (too many incompatible dependencies), the fallback is **ReadyToRun (R2R)** compilation (`<PublishReadyToRun>true</PublishReadyToRun>`). R2R pre-compiles methods to native code at publish time but still ships the IL and runtime — the JIT can re-compile hot paths with better optimizations at runtime. R2R reduces cold start by ~50–60% (not to zero), has broader compatibility, and is a good middle ground. You can also combine R2R with **tiered compilation** and `<TieredPGO>true</TieredPGO>` for steady-state performance.

**Hint:** A strong answer distinguishes Native AOT (no JIT at all, maximum cold-start reduction, most restrictions) from ReadyToRun (partial pre-compilation, fewer restrictions). Follow-up: "What is trimming and why does AOT require it?" (Trimming removes unreferenced code to reduce binary size; AOT needs it because it must know the full call graph at compile time.) "What about crossgen2 vs Native AOT?" (crossgen2 produces R2R images; Native AOT uses ILC to produce fully native binaries.)

**🚩 Red Signal:** Doesn't know about Native AOT or ReadyToRun, suggests "just add more RAM" or "pre-warm with a health check endpoint" (pre-warming doesn't help in true serverless cold starts where the process doesn't exist yet).
