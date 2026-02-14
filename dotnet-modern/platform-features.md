# Platform Features

---

### 1. 🟢 You're migrating a .NET 5 project that uses `Startup.cs` with `ConfigureServices` and `Configure` methods to .NET 8. Walk through what changes in the hosting model.

In .NET 6+, the `Startup.cs` class is replaced by a single `Program.cs` using `WebApplicationBuilder` and `WebApplication`. Service registration that was in `ConfigureServices` moves to `builder.Services`, and middleware configuration that was in `Configure` moves to `app.Use*()` calls — all in a linear, top-to-bottom flow. The middleware pipeline order still matters and must follow the correct sequence: `UseExceptionHandler` → `UseHsts` → `UseRouting` → `UseAuthentication` → `UseAuthorization` → endpoint mapping. Top-level statements remove the `Main` method boilerplate, and `builder.Configuration` replaces the `IConfiguration` injection in `Startup`.

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();
builder.Services.AddScoped<IOrderRepository, SqlOrderRepository>();

var app = builder.Build();

app.UseExceptionHandler("/error");
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
app.Run();
```

**Hint:** The candidate should understand the middleware pipeline order and why it matters. Follow up: "What breaks if you put `UseAuthorization` before `UseAuthentication`?" and "Can you still use a Startup class if you prefer?"

**🚩 Red Signal:** Cannot explain the middleware pipeline order, or puts `UseAuthorization` before `UseAuthentication`.

---

### 2. 🟡 Your team is evaluating whether to upgrade from .NET 6 to .NET 8. What concrete improvements justify the effort?

.NET 8 brings several tangible improvements worth the upgrade cost. Native AOT support enables ahead-of-time compilation for faster startup and smaller binaries in containerised workloads. The Request Delegate Generator source-generates minimal API request delegates, removing runtime reflection overhead. Frozen collections (`FrozenDictionary<K,V>`, `FrozenSet<T>`) optimise read-heavy lookup scenarios. `TimeProvider` introduces a testable time abstraction, replacing `DateTime.UtcNow` calls that are hard to mock. `IExceptionHandler` provides a cleaner global exception handling pattern in the middleware pipeline. `System.Text.Json` gains better source generator support and performance. Keyed services in DI allow injecting different implementations of the same interface by key. The upgrade also brings long-term support (LTS) status, meaning three years of security patches compared to .NET 6's end-of-life timeline.

**Hint:** Look for practical experience — has the candidate actually used any of these features in a real project? Follow up: "Which of these had the biggest impact on your team, and why?" Bonus for discussing trade-offs such as Native AOT limitations with EF Core and reflection.

**🚩 Red Signal:** Unaware of any specific improvements beyond "it's faster," or has not kept up with .NET releases since .NET 6.

---

### 3. 🔴 You're deploying a .NET API to a serverless platform with strict cold-start requirements. JIT warmup adds 2 seconds to the first request. How do you eliminate that latency?

Native AOT compilation is the primary solution. It compiles .NET code directly to native machine code at build time, completely eliminating the JIT compiler at runtime. This yields sub-100ms startup times, lower memory footprint, and smaller deployment binaries — ideal for serverless functions, CLI tools, and high-density container workloads. However, AOT has significant constraints: no `Reflection.Emit` or dynamic assembly loading, limited reflection support requiring trimming annotations, and not all NuGet packages are AOT-compatible. EF Core has limited AOT support, so data access in AOT scenarios typically uses Dapper or raw ADO.NET. For long-running services that benefit from tiered JIT compilation and profile-guided optimisation, standard JIT is often faster at steady state — AOT trades peak throughput for startup speed.

```xml
<PropertyGroup>
    <PublishAot>true</PublishAot>
</PropertyGroup>
```

**Hint:** A strong candidate explains *when* AOT makes sense (serverless, CLI, high-density microservices) vs when JIT is better (long-running services benefiting from tiered compilation). Follow up: "How do you verify that your dependencies are AOT-compatible before committing to this approach?" and "What's the difference between trimming and full AOT?"

**🚩 Red Signal:** Assumes AOT works with every .NET library, is unaware of reflection limitations, or dismisses startup time as unimportant for serverless.

---

### 4. 🟢 A junior developer on your team is calling `app.Services.GetService<IOrderRepository>()` inside minimal API handlers instead of accepting it as a parameter. What's wrong with this approach, and how should it work?

Calling `GetService<T>()` manually is the service locator anti-pattern — it hides dependencies, makes the code harder to test, and bypasses the DI container's lifetime management for that scope. In minimal APIs, the correct approach is parameter injection: the handler declares its dependencies as method parameters, and the DI container resolves them automatically. This makes dependencies explicit, enables proper scoping (a `Scoped` service gets the correct per-request instance), and simplifies unit testing since you can pass mock implementations directly. .NET 8 also introduces keyed services (`AddKeyedScoped<T>`) for injecting different implementations of the same interface based on a key.

```csharp
// ❌ Service locator — hidden dependency, wrong scope
app.MapGet("/orders/{id}", (int id, HttpContext ctx) =>
{
    var repo = ctx.RequestServices.GetService<IOrderRepository>();
    return repo.GetByIdAsync(id);
});

// ✅ Parameter injection — explicit, testable, correct lifetime
app.MapGet("/orders/{id}", async (int id, IOrderRepository repo) =>
    await repo.GetByIdAsync(id) is { } order
        ? Results.Ok(order)
        : Results.NotFound());
```

**Hint:** Look for understanding of why service locator is an anti-pattern (hidden dependencies, testability, lifetime issues). Follow up: "What are keyed services in .NET 8, and when would you use them?"

**🚩 Red Signal:** Sees nothing wrong with `GetService<T>()`, or does not understand how minimal API parameter injection works.

---

### 5. 🟡 A developer registered `DbContext` as Scoped and injected it into a Singleton background service. Users are reporting stale data, and the app occasionally throws `ObjectDisposedException`. What went wrong, and how do you fix it?

This is the captive dependency problem. A `Scoped` service (DbContext, created per-request) was captured by a `Singleton` service (created once for the application lifetime). The Singleton holds a reference to the first DbContext instance forever — it never gets disposed or replaced. Subsequent requests see stale data from the original context's change tracker, and when the original scope is eventually disposed, the DbContext throws `ObjectDisposedException`. The fix is to inject `IServiceScopeFactory` into the Singleton and create a new scope each time the service needs a DbContext. In .NET 8, `ValidateScopes` is enabled by default in Development mode, which catches this mistake at startup with a clear exception.

```csharp
public class OrderProcessingService : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;

    public OrderProcessingService(IServiceScopeFactory scopeFactory)
        => _scopeFactory = scopeFactory;

    protected override async Task ExecuteAsync(CancellationToken token)
    {
        using var scope = _scopeFactory.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        // db is correctly scoped and will be disposed with the scope
    }
}
```

| Lifetime | Instance per | Typical use |
|---|---|---|
| Singleton | Application | Caches, configuration, `HttpClient` factories |
| Scoped | HTTP request | `DbContext`, unit-of-work |
| Transient | Injection | Lightweight, stateless services |

**Hint:** The candidate should explain captive dependency clearly and know that `ValidateScopes` catches it in Development. Follow up: "What happens if you register the DbContext as Transient instead? Does that fix it?" and "How does `ValidateScopes` work under the hood?"

**🚩 Red Signal:** Registers everything as Singleton for perceived performance, or does not understand the captive dependency problem.

---

### 6. 🟡 Kubernetes keeps restarting your API pods even though the application process is running. You have no health checks configured. How do you set up liveness and readiness probes to fix this?

Kubernetes uses liveness probes to detect if the process is hung (and should be killed) and readiness probes to detect if the pod can serve traffic (dependencies are up). Without health check endpoints, Kubernetes falls back to process-level checks that only know if the process is alive — not whether it can actually handle requests. You configure ASP.NET Core health checks by registering them in DI with `AddHealthChecks()`, adding checks for dependencies like SQL Server and Redis, and mapping two separate endpoints. The liveness endpoint (`/health/live`) should return healthy if the app is running — no dependency checks. The readiness endpoint (`/health/ready`) should verify that databases, caches, and external services are reachable. In the Kubernetes deployment manifest, point the `livenessProbe` at `/health/live` and the `readinessProbe` at `/health/ready`.

```csharp
builder.Services.AddHealthChecks()
    .AddSqlServer(connectionString, tags: new[] { "ready" })
    .AddRedis(redisConnection, tags: new[] { "ready" })
    .AddCheck<CustomHealthCheck>("custom", tags: new[] { "ready" });

app.MapHealthChecks("/health/ready", new() { Predicate = check => check.Tags.Contains("ready") });
app.MapHealthChecks("/health/live",  new() { Predicate = _ => false }); // liveness = app process is running
```

**Hint:** The candidate should clearly explain the difference between liveness (app not crashed/hung) and readiness (app can serve requests — dependencies are up). Follow up: "What happens if you include a database check in the liveness probe?" and "How would you add a startup probe for slow-initialising services?"

**🚩 Red Signal:** Has no health check strategy, conflates liveness with readiness, or puts dependency checks on the liveness probe (causing restart loops when a database is temporarily down).
