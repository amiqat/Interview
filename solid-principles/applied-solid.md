# Applied SOLID

Advanced questions on applying SOLID principles in real systems — trade-offs, patterns, and knowing when to bend the rules.

---

### 1. 🔴 You're designing a background job processing system that handles email sending, report generation, and data imports. How do you apply SOLID principles to its architecture?

- **SRP:** Separate job scheduling, execution, retry logic, and dead-letter handling into distinct classes.
- **OCP:** New job types implement `IJob` or `IJobHandler<T>` — no changes to the processor/dispatcher.
- **DIP:** The dispatcher depends on `IJobHandler<T>`, resolved from DI; concrete handlers depend on injected repository/service abstractions.
- **LSP:** All `IJobHandler<T>` implementations honour the contract (return success/failure, do not throw unhandled exceptions).
- **ISP:** If some jobs need retry logic and others do not, separate `IRetryableJob` from `IJob`.

**Hint:** A great candidate draws parallels to libraries like MediatR, Hangfire, or MassTransit and explains how those frameworks already embody SOLID principles.

**🚩 Red Signal:** Puts all job logic in a single `BackgroundService` with a switch statement for each job type.

---

### 2. 🔴 Your Singleton `NotificationService` takes a Scoped `DbContext` via constructor injection. In production, you're seeing stale data and occasional threading exceptions. What's wrong?

A captive dependency occurs when a service with a shorter lifetime is injected into a service with a longer lifetime. A `Scoped` `DbContext` injected into a `Singleton` service is captured and reused across all requests, causing threading issues and stale data.

**Fix:** Inject `IServiceScopeFactory` into the Singleton and create a scope per operation, or redesign the lifetimes.

**Hint:** ASP.NET Core 8+ throws by default (`ValidateScopes` is on in Development). The candidate should know this behaviour.

**🚩 Red Signal:** Registers everything as Singleton "for performance" without considering lifetime mismatches.

---

### 3. 🟡 A junior developer creates an interface for every single class, even those that will never have a second implementation. They say it's "following SOLID." How do you respond?

SOLID principles are guidelines, not laws. Over-engineering signs:
- Creating interfaces for classes that will never have a second implementation.
- Abstracting prematurely before understanding the domain.
- Adding layers of indirection that make the code harder to follow.

Pragmatic approach: apply SOLID where change is expected or where testability requires it. Use the **Rule of Three** — refactor to an abstraction when you see the same pattern for the third time.

**Hint:** A mature developer balances SOLID with YAGNI (You Ain't Gonna Need It) and KISS (Keep It Simple, Stupid). Look for nuanced thinking rather than dogmatic adherence.

**🚩 Red Signal:** Either applies every principle religiously to trivial code, or dismisses SOLID entirely as "academic nonsense."

---

### 4. 🟡 Your team needs to add new discount types frequently. The current `OrderService` has a growing chain of `if` checks for each discount. How would you redesign this using a pattern that respects OCP? Show the C# code.

```csharp
public interface IDiscountStrategy
{
    decimal Calculate(Order order);
}

public class SeasonalDiscount : IDiscountStrategy
{
    public decimal Calculate(Order order) => order.Total * 0.1m;
}

public class LoyaltyDiscount : IDiscountStrategy
{
    public decimal Calculate(Order order) => order.Total * 0.15m;
}

public class OrderService
{
    private readonly IDiscountStrategy _discount;
    public OrderService(IDiscountStrategy discount) => _discount = discount;
    public decimal GetFinalPrice(Order order) => order.Total - _discount.Calculate(order);
}
```

Adding a new discount type requires only a new class — `OrderService` is closed for modification.

**Hint:** Bonus if the candidate discusses how DI registration (`services.AddScoped<IDiscountStrategy, SeasonalDiscount>()`) selects the strategy, or uses a factory/dictionary for runtime strategy selection.

**🚩 Red Signal:** Cannot connect OCP to any concrete design pattern or implementation approach.

---

### 5. 🔴 A tech lead argues that for a simple CRUD endpoint, creating separate validator, handler, mapper, and repository classes is overkill. When would you intentionally violate a SOLID principle and how do you justify it?

Examples of justified violations:
- **SRP:** A simple CRUD controller that handles validation, mapping, and persistence for a straightforward entity — the cost of abstraction outweighs the benefit.
- **OCP:** Modifying an existing class when adding an extension point would be over-engineering for a one-off change.
- **DIP:** Using a concrete class directly when it is a stable dependency unlikely to change (e.g., `List<T>`, `StringBuilder`).

**Hint:** The best candidates evaluate trade-offs: team size, project lifespan, rate of change, and complexity budget. They apply SOLID where it delivers value.

**🚩 Red Signal:** Either cannot imagine any scenario where violating SOLID is acceptable, or uses this as an excuse to never follow any principle.
