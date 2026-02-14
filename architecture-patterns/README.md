# Architecture & Design Patterns

Questions on software architecture patterns, design patterns, and architectural decision-making for .NET applications.

---

### 1. 🟡 Explain the Clean Architecture pattern. How do you implement it in a .NET solution?

Clean Architecture organises code into concentric layers with the **domain** at the centre and **dependencies pointing inward**:

- **Domain** — Entities, value objects, domain events, interfaces. No external dependencies.
- **Application** — Use cases, DTOs, validation. Depends only on Domain.
- **Infrastructure** — EF Core, external APIs, file system. Implements interfaces from Application/Domain.
- **Presentation** — Controllers, minimal API endpoints. Depends on Application.

**Hint:** Look for understanding that the key rule is the **Dependency Rule** — outer layers depend on inner layers, never the reverse. The candidate should also discuss when Clean Architecture is overkill (small CRUD apps).

**🚩 Red Signal:** Cannot explain the dependency direction, or applies Clean Architecture to every project regardless of complexity.

---

### 2. 🔴 When would you choose vertical slice architecture over layered architecture?

**Layered (horizontal):** Organise by technical concern — Controllers, Services, Repositories. Changes to a feature touch multiple layers.

**Vertical slice:** Organise by feature — each slice contains the handler, validation, data access, and response for a single use case. Libraries like MediatR enable this with `IRequest<T>` / `IRequestHandler<T, R>`.

**When to choose vertical slices:** Feature-heavy applications where each feature has distinct behaviour, teams that want to minimise merge conflicts, and codebases where layers have become overly generic.

**Hint:** A principal-level candidate doesn't dogmatically choose one — they explain the trade-offs and pick based on team size, project complexity, and rate of change.

**🚩 Red Signal:** Has only ever used one pattern and cannot articulate alternatives.

---

### 3. 🟡 What is the Mediator pattern and how does MediatR implement it in .NET?

The Mediator pattern decouples senders from receivers by routing requests through a central mediator. MediatR implements this:

- **`IRequest<T>`** — A request (command or query).
- **`IRequestHandler<TRequest, TResponse>`** — Handles exactly one request type.
- **Pipeline behaviours** — Cross-cutting concerns (logging, validation, transaction management) implemented as `IPipelineBehavior<,>`.

**Hint:** Good answers discuss the trade-off: MediatR adds indirection (harder to "Go to Definition") in exchange for decoupling and a clean pipeline. The candidate should have an opinion.

**🚩 Red Signal:** Uses MediatR everywhere (even for simple CRUD) without being able to explain why, or has no awareness of the "in-process only" limitation.

---

### 4. 🔴 Explain CQRS (Command Query Responsibility Segregation). When is it worth the complexity?

CQRS separates the **write model** (commands — optimised for consistency and validation) from the **read model** (queries — optimised for fast retrieval, possibly denormalised).

**Worth it when:** Read and write patterns are very different (e.g., complex writes but simple flat reads), high read:write ratio, or when combined with event sourcing.

**Not worth it when:** Simple CRUD with similar read/write shapes — CQRS adds unnecessary complexity.

**Hint:** A principal-level candidate should be able to explain CQRS without event sourcing (they are independent patterns) and discuss the consistency challenge of a separate read store.

**🚩 Red Signal:** Conflates CQRS with event sourcing, or implements CQRS for a basic CRUD application.

---

### 5. 🟡 What is the Repository pattern? Is it still useful with EF Core?

The Repository pattern abstracts data access behind an interface (`IOrderRepository`), hiding the persistence mechanism from the business logic.

**Arguments for:** Testability (mock the repository), swappable data stores, consistent query patterns.

**Arguments against with EF Core:** `DbContext` already implements Unit of Work + Repository (`DbSet<T>`). Adding a repository on top creates a "repository over repository" that limits EF Core features (LINQ composition, change tracking).

**Hint:** A principal-level candidate has a nuanced opinion. Common pragmatic approaches: use repositories for complex aggregates, skip them for simple queries; or use a thin "query service" instead.

**🚩 Red Signal:** Always wraps `DbContext` in a generic repository that returns `IQueryable<T>` (leaking the abstraction), or dismisses all patterns as unnecessary.

---

### 6. 🟡 How do you implement the Options pattern for configuration in ASP.NET Core?

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

### 7. 🟢 What is the Unit of Work pattern and how does EF Core implement it?

The Unit of Work pattern tracks all changes made during a business transaction and commits them atomically. `DbContext` is the Unit of Work in EF Core: it tracks entity changes and `SaveChanges()` writes them all in a single database transaction.

**Hint:** The candidate should explain why calling `SaveChanges()` once at the end (not after each operation) ensures atomicity, and how `IDbContextTransaction` enables explicit transaction control across multiple `SaveChanges()` calls.

**🚩 Red Signal:** Calls `SaveChanges()` after every single entity change, or creates a new `DbContext` per operation within the same request.

---

### 8. 🟡 How do you implement the Circuit Breaker pattern for external service calls?

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

### 9. 🟡 Explain the Decorator pattern and how ASP.NET Core's DI supports it.

The Decorator pattern wraps an existing implementation to add behaviour without modifying it. In .NET DI, use `Scrutor` or manual registration:

```csharp
// Manual decoration
builder.Services.AddScoped<IOrderRepository, SqlOrderRepository>();
builder.Services.Decorate<IOrderRepository, CachingOrderRepository>();
builder.Services.Decorate<IOrderRepository, LoggingOrderRepository>();
```

The call chain: `LoggingOrderRepository` → `CachingOrderRepository` → `SqlOrderRepository`.

**Hint:** This is a practical application of OCP — add behaviour by wrapping, not modifying. The candidate should explain the use case: caching, logging, retry, validation.

**🚩 Red Signal:** Modifies the original class to add caching/logging instead of decorating, or has never used the pattern.

---

### 10. 🔴 You need to design a plugin system where new features can be added without redeploying the main application. How?

**Approaches:**
- **Assembly loading:** `AssemblyLoadContext` to load plugins at runtime from a folder. Define a shared `IPlugin` interface in a contracts package.
- **Configuration-driven:** Feature flags and strategy patterns where new strategies are registered via configuration.
- **Module system:** Each module is a NuGet package with its own endpoints, services, and migrations. Compose at startup.

**Hint:** Look for understanding of isolation (separate `AssemblyLoadContext` for unloading), versioning (contract stability), and security (running untrusted code).

**🚩 Red Signal:** Suggests hard-coding all features and redeploying for every change, or has no concept of runtime extensibility.
