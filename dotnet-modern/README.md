# .NET Modern — .NET 8/10, Minimal APIs & New Features

Questions on modern .NET features, minimal APIs vs controllers, middleware pipeline, and framework evolution for .NET 8 and beyond.

---

### 1. What are minimal APIs in .NET and how do they differ from controller-based APIs?

Minimal APIs define endpoints directly in `Program.cs` using `app.MapGet`, `app.MapPost`, etc. — no controllers, no `[ApiController]` attribute, no model binding conventions by default.

| | Minimal APIs | Controllers |
|---|---|---|
| Setup | Inline in `Program.cs` or extension methods | Class with `[ApiController]` attribute |
| Routing | `app.MapGet("/orders", handler)` | `[Route("api/[controller]")]` + `[HttpGet]` |
| Filters | Endpoint filters (`AddEndpointFilter`) | Action filters, result filters |
| Model binding | Explicit via `[FromBody]`, `[FromQuery]` | Convention-based (automatic) |
| Best for | Microservices, small APIs, performance-critical | Large APIs, teams familiar with MVC patterns |

**Hint:** A principal-level candidate discusses *when* to use each — not just *how*. Minimal APIs suit small, focused services; controllers suit complex APIs with many cross-cutting concerns. Both can coexist in the same project.

**🚩 Red Signal:** Dismisses minimal APIs as "just for demos" or cannot explain the differences beyond syntax.

---

### 2. How do you organise minimal API endpoints as the project grows beyond a few routes?

As `Program.cs` grows, structure with:
- **Extension methods:** `app.MapOrderEndpoints()` in a separate `OrderEndpoints.cs` file.
- **Carter library:** Convention-based endpoint modules.
- **Route groups:** `app.MapGroup("/api/orders").MapGet(...)` to share prefixes, filters, and metadata.
- **Vertical slices:** Group endpoints by feature (not by layer).

**Hint:** Look for awareness that minimal APIs do not enforce structure — the developer must be intentional about organisation. A good answer reflects real project experience.

**🚩 Red Signal:** Puts all 50+ endpoints in `Program.cs` and sees no problem with it.

---

### 3. What are endpoint filters in minimal APIs and how do they compare to MVC action filters?

Endpoint filters (`AddEndpointFilter`) are the minimal API equivalent of action/result filters. They wrap the endpoint handler and can execute logic before and after the handler runs (validation, logging, authorization).

```csharp
app.MapPost("/orders", CreateOrder)
   .AddEndpointFilter(async (context, next) =>
   {
       // Before: validate
       var result = await next(context);
       // After: log
       return result;
   });
```

**Hint:** Candidates should know that endpoint filters are invoked in registration order (pipeline), and that `IEndpointFilter` can be implemented as a class for reusability.

**🚩 Red Signal:** Unaware that minimal APIs have any middleware/filter mechanism.

---

### 4. Explain the `IResult` return type in minimal APIs. How does it compare to `IActionResult`?

`IResult` is the minimal API equivalent of `IActionResult`. Built-in factory methods: `Results.Ok()`, `Results.NotFound()`, `Results.Created()`, `Results.Problem()`.

In .NET 7+, **typed results** (`Results<Ok<Order>, NotFound>`) enable OpenAPI metadata generation without manual `[ProducesResponseType]` attributes.

**Hint:** A strong answer covers `TypedResults` (static class with typed return values) vs `Results` (non-typed), and how typed results improve Swagger documentation automatically.

**🚩 Red Signal:** Returns raw objects without status codes or uses `Results.Ok()` for everything including errors.

---

### 5. What is the `WebApplication` and `WebApplicationBuilder` pattern introduced in .NET 6+?

`WebApplicationBuilder` replaces `Startup.cs` with a single `Program.cs` that configures services and middleware in a linear flow. `builder.Services` registers DI services, `builder.Build()` creates the `WebApplication`, and `app.Use*()` configures the middleware pipeline.

**Hint:** The candidate should understand the middleware pipeline order and why it matters: `UseExceptionHandler` → `UseHsts` → `UseRouting` → `UseAuthentication` → `UseAuthorization` → endpoint mapping.

**🚩 Red Signal:** Cannot explain the middleware pipeline or puts `UseAuthorization` before `UseAuthentication`.

---

### 6. What are the key performance improvements in .NET 8 compared to earlier versions?

- **Native AOT** — Ahead-of-time compilation for faster startup and smaller binaries. Limited to a subset of APIs (no reflection-heavy code).
- **Request Delegate Generator** — Source-generated request delegates for minimal APIs, replacing runtime reflection.
- **Frozen collections** — `FrozenDictionary<K,V>` / `FrozenSet<T>` optimised for read-heavy scenarios.
- **`TimeProvider`** — Abstraction for testable time-dependent code.
- **`IExceptionHandler`** — New interface for global exception handling in the middleware pipeline.
- **Improved `System.Text.Json`** — Source generators, `JsonSerializerOptions.Default`, better performance.

**Hint:** Look for practical experience — has the candidate actually used any of these in a real project? Bonus for discussing trade-offs (e.g., Native AOT limitations with EF Core and reflection).

**🚩 Red Signal:** Unaware of any improvements beyond "it's faster" or has not upgraded from .NET 6.

---

### 7. How does Native AOT compilation work and what are its limitations?

Native AOT compiles .NET code directly to native machine code at build time, eliminating the JIT compiler at runtime. This yields faster startup, lower memory footprint, and smaller deployment size.

**Limitations:**
- No runtime code generation (`Reflection.Emit`, dynamic assembly loading).
- Limited reflection support (requires trimming annotations).
- Not all NuGet packages are AOT-compatible.
- EF Core has limited AOT support (use Dapper or raw ADO.NET for AOT scenarios).

**Hint:** A principal-level candidate explains *when* AOT makes sense (serverless functions, CLI tools, high-density microservices) vs when the JIT is better (long-running services that benefit from tiered compilation).

**🚩 Red Signal:** Assumes AOT works with every .NET library or dismisses it as unnecessary.

---

### 8. How do you configure dependency injection in a .NET 8 minimal API application?

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddScoped<IOrderRepository, SqlOrderRepository>();
builder.Services.AddSingleton<ICacheService, RedisCacheService>();
builder.Services.AddHttpClient<IPaymentClient, PaymentClient>();

var app = builder.Build();

app.MapGet("/orders/{id}", async (int id, IOrderRepository repo) =>
    await repo.GetByIdAsync(id) is { } order
        ? Results.Ok(order)
        : Results.NotFound());
```

Minimal API handlers receive dependencies as method parameters — no constructor injection needed. The DI container resolves them automatically.

**Hint:** Look for understanding of `Keyed services` (.NET 8), `AddKeyedScoped<T>`, and how to inject different implementations of the same interface in different endpoints.

**🚩 Red Signal:** Manually resolves services with `app.Services.GetService<T>()` instead of using parameter injection.

---

### 9. What is the difference between `AddSingleton`, `AddScoped`, and `AddTransient`? Give a scenario where using the wrong lifetime causes a bug.

| Lifetime | Instance per | Typical use |
|---|---|---|
| Singleton | Application | Caches, configuration, `HttpClient` factories |
| Scoped | HTTP request | `DbContext`, unit-of-work |
| Transient | Injection | Lightweight, stateless services |

**Bug scenario (captive dependency):** A `Scoped` `DbContext` injected into a `Singleton` service. The DbContext is captured once and reused across all requests — stale data, threading issues, eventual `ObjectDisposedException`.

**Hint:** In .NET 8, `ValidateScopes` is on by default in Development, catching this at startup. A principal-level candidate should know this.

**🚩 Red Signal:** Registers everything as `Singleton` for perceived performance, or does not understand the captive dependency problem.

---

### 10. How do you implement health checks and what monitoring patterns should a production .NET 8 API have?

```csharp
builder.Services.AddHealthChecks()
    .AddSqlServer(connectionString)
    .AddRedis(redisConnection)
    .AddCheck<CustomHealthCheck>("custom");

app.MapHealthChecks("/health/ready", new() { Predicate = check => check.Tags.Contains("ready") });
app.MapHealthChecks("/health/live",  new() { Predicate = _ => false }); // liveness = app running
```

**Production monitoring:** Health check endpoints for Kubernetes probes (liveness + readiness), structured logging (Serilog → Seq/ELK), distributed tracing (OpenTelemetry), metrics (Prometheus/Grafana).

**Hint:** A principal-level developer should explain the difference between liveness (app not crashed) and readiness (app can serve requests — dependencies are up) probes.

**🚩 Red Signal:** Has no health check or monitoring strategy, or relies solely on try/catch-based error logging.
