# Design Patterns

Practical design patterns for .NET applications — how and when to apply them.

---

### 1. 🟢 What is the Unit of Work pattern and how does EF Core implement it?

The Unit of Work pattern tracks all changes made during a business operation and commits them atomically in a single transaction. EF Core's `DbContext` is the Unit of Work — it tracks entity changes and `SaveChanges()` wraps everything in one database transaction.

**Hint:** The candidate should explain why calling `SaveChanges()` once at the end (not after each operation) ensures atomicity.

**🚩 Red Signal:** Cannot explain what Unit of Work means or how `DbContext` implements it.

---

### 2. 🟡 Your controllers are bloated — 50+ lines of validation, DB calls, and mapping per action. How would you decouple the request handling from the controller?

The Mediator pattern decouples senders from receivers by routing requests through a central mediator. MediatR implements this in .NET:

- **`IRequest<T>`** — A request (command or query).
- **`IRequestHandler<TRequest, TResponse>`** — Handles exactly one request type.
- **Pipeline behaviours** — Cross-cutting concerns (logging, validation, transaction management) implemented as `IPipelineBehavior<,>`.

The controller becomes a thin routing layer that dispatches requests to handlers. Each handler is a focused, testable class.

**Hint:** Good answers discuss the trade-off: MediatR adds indirection (harder to "Go to Definition") in exchange for decoupling and a clean pipeline. The candidate should have an opinion.

**🚩 Red Signal:** Uses MediatR everywhere (even for simple CRUD) without being able to explain why, or has no awareness of the "in-process only" limitation.

---

### 3. 🟡 You're reading `Configuration["Payment:ApiKey"]` as magic strings in 15 places across the codebase. How do you clean this up? What are the differences between `IOptions<T>`, `IOptionsSnapshot<T>`, and `IOptionsMonitor<T>`?

```csharp
// appsettings.json
{ "Payment": { "ApiKey": "...", "TimeoutSeconds": 30 } }

// Strongly-typed class
public class PaymentOptions { public string ApiKey { get; set; } public int TimeoutSeconds { get; set; } }

// Registration
builder.Services.Configure<PaymentOptions>(builder.Configuration.GetSection("Payment"));

// Injection
public class PaymentService(IOptions<PaymentOptions> options) { ... }
```

**`IOptions<T>` vs `IOptionsSnapshot<T>` vs `IOptionsMonitor<T>`:**
- `IOptions<T>` — Singleton, read once at startup.
- `IOptionsSnapshot<T>` — Scoped, refreshed per request.
- `IOptionsMonitor<T>` — Singleton but supports change notifications.

**Hint:** The candidate should know about `ValidateDataAnnotations()` and `ValidateOnStart()` (.NET 8) for fail-fast configuration validation.

**🚩 Red Signal:** Reads `Configuration["Payment:ApiKey"]` as magic strings everywhere instead of using strongly-typed options.

---

### 4. 🟢 A developer calls `SaveChanges()` after each entity update within a single request — resulting in 5 separate database transactions. Is this a problem? How should it be handled?

The Unit of Work pattern tracks all changes made during a business transaction and commits them atomically. `DbContext` is the Unit of Work in EF Core: it tracks entity changes and `SaveChanges()` writes them all in a single database transaction.

Calling `SaveChanges()` five times means five separate transactions — if the third one fails, the first two are already committed and cannot be rolled back. The fix is to make all changes and call `SaveChanges()` once at the end, ensuring atomicity.

**Hint:** The candidate should explain why calling `SaveChanges()` once at the end (not after each operation) ensures atomicity, and how `IDbContextTransaction` enables explicit transaction control across multiple `SaveChanges()` calls.

**🚩 Red Signal:** Calls `SaveChanges()` after every single entity change, or creates a new `DbContext` per operation within the same request.

---

### 5. 🟡 Your payment API is failing intermittently — timeouts and 500 errors. Your retry logic keeps hammering it even when it's clearly down. How do you stop calling a failing service and recover gracefully?

Using Polly (standalone) or `Microsoft.Extensions.Http.Resilience` (.NET 8):

```csharp
builder.Services.AddHttpClient<IPaymentClient, PaymentClient>()
    .AddResilienceHandler("payment", builder =>
    {
        builder.AddCircuitBreaker(new CircuitBreakerStrategyOptions<HttpResponseMessage>
        {
            FailureRatio = 0.5,
            SamplingDuration = TimeSpan.FromSeconds(30),
            BreakDuration = TimeSpan.FromSeconds(15),
            MinimumThroughput = 10
        });
        builder.AddRetry(new HttpRetryStrategyOptions { MaxRetryAttempts = 3 });
        builder.AddTimeout(TimeSpan.FromSeconds(5));
    });
```

**States:** Closed (normal) → Open (failing, reject calls) → Half-Open (test with one request).

**Hint:** The candidate should explain the order: timeout → retry → circuit breaker (innermost to outermost), and why monitoring circuit breaker state changes is critical.

**🚩 Red Signal:** Retries indefinitely without a circuit breaker, or has no resilience strategy for external calls.

---

### 6. 🟡 You need to add caching and logging to `SqlOrderRepository` without modifying it. How do you layer on this behaviour?

The Decorator pattern wraps an existing implementation to add behaviour without modifying it. In .NET DI, use `Scrutor` or manual registration:

```csharp
// Manual decoration
builder.Services.AddScoped<IOrderRepository, SqlOrderRepository>();
builder.Services.Decorate<IOrderRepository, CachingOrderRepository>();
builder.Services.Decorate<IOrderRepository, LoggingOrderRepository>();
```

The call chain: `LoggingOrderRepository` → `CachingOrderRepository` → `SqlOrderRepository`.

This is a practical application of OCP — add behaviour by wrapping, not modifying. Each decorator has a single responsibility (caching or logging) and can be composed in any order.

**Hint:** The candidate should explain the use case: caching, logging, retry, validation — all without touching the original class.

**🚩 Red Signal:** Modifies the original class to add caching/logging instead of decorating, or has never used the pattern.

---

### 7. 🔴 Your application needs a plugin system — new features must be added without redeploying the main application. How do you design this?

**Approaches:**
- **Assembly loading:** `AssemblyLoadContext` to load plugins at runtime from a folder. Define a shared `IPlugin` interface in a contracts package.
- **Configuration-driven:** Feature flags and strategy patterns where new strategies are registered via configuration.
- **Module system:** Each module is a NuGet package with its own endpoints, services, and migrations. Compose at startup.

**Hint:** Look for understanding of isolation (separate `AssemblyLoadContext` for unloading), versioning (contract stability), and security (running untrusted code).

**🚩 Red Signal:** Suggests hard-coding all features and redeploying for every change, or has no concept of runtime extensibility.
