# SOLID Principles

Questions focused on the Single Responsibility, Open/Closed, and Dependency Inversion principles, with coverage of Liskov Substitution and Interface Segregation.

---

### 1. Explain the Single Responsibility Principle (SRP). Give a real-world .NET example of a violation and how you would fix it.

SRP states that a class should have only one reason to change — it should encapsulate one responsibility. A common violation is a `UserService` that handles user registration, sends emails, writes audit logs, and validates input all in one class.

**Fix:** Extract responsibilities into focused classes: `UserRegistrationService`, `IEmailSender`, `IAuditLogger`, `IUserValidator`. Each class has one axis of change.

**Hint:** Look for candidates who talk about *cohesion* — methods and data that belong together. SRP is not "one method per class"; it is about having a single reason to change.

**🚩 Red Signal:** Interprets SRP as "each class should have only one method" or cannot identify when a class has too many responsibilities.

---

### 2. How does the Open/Closed Principle (OCP) apply when adding a new payment method to an e-commerce system?

OCP states that software entities should be open for extension but closed for modification. Instead of adding a new `if`/`switch` branch to a `ProcessPayment` method for every new provider, define an `IPaymentProcessor` interface and create a new implementation (`StripePaymentProcessor`, `PayPalPaymentProcessor`). Register them in the DI container and resolve via a factory or strategy pattern.

**Hint:** Great answers mention that the DI container and the strategy/factory pattern are the practical mechanisms for achieving OCP in .NET. Also look for awareness that OCP does not mean *never* modifying existing code — it means designing extension points where change is expected.

**🚩 Red Signal:** Adds every new feature by modifying a giant `switch` statement, or claims you should never modify any existing code under any circumstance.

---

### 3. Explain the Dependency Inversion Principle (DIP) and how it relates to Dependency Injection in ASP.NET Core.

DIP states: (A) High-level modules should not depend on low-level modules — both should depend on abstractions. (B) Abstractions should not depend on details — details should depend on abstractions.

In ASP.NET Core, this is realised through the built-in DI container: controllers (high-level) depend on interfaces (`IOrderRepository`), and the DI container wires in concrete implementations (`SqlOrderRepository`). The controller never knows about SQL Server.

**Hint:** A strong candidate distinguishes DIP (the principle) from DI (the pattern) from IoC containers (the tool). They should also explain service lifetimes: `Transient`, `Scoped`, `Singleton` — and the captive dependency problem (a Scoped service injected into a Singleton).

**🚩 Red Signal:** Equates DIP with "using an IoC container" without understanding why abstractions matter, or creates interfaces for every class without purpose (over-abstraction).

---

### 4. What is the Liskov Substitution Principle (LSP)? Give an example of a violation in C#.

LSP states that objects of a derived class should be substitutable for objects of the base class without altering the correctness of the program. The classic violation: `Square` inherits from `Rectangle` and overrides `Width`/`Height` setters to keep them equal — code that expects independent width/height breaks.

In .NET: a `ReadOnlyCollection<T>` that inherits from a mutable `IList<T>` and throws `NotSupportedException` on `Add()` violates LSP because callers of `IList<T>` expect `Add` to work.

**Hint:** LSP is about *behavioural* compatibility, not just method signatures. Look for candidates who can discuss preconditions, postconditions, and invariants.

**🚩 Red Signal:** Has never heard of LSP or thinks it is only about method signature compatibility.

---

### 5. How does the Interface Segregation Principle (ISP) help in designing .NET services?

ISP states that clients should not be forced to depend on interfaces they do not use. Instead of a fat `IUserService` with `Register`, `Login`, `UpdateProfile`, `DeleteAccount`, `GenerateReport`, split into focused interfaces: `IUserRegistration`, `IUserAuthentication`, `IUserProfileManager`.

**Hint:** In practice, ISP aligns with SRP at the interface level. Look for understanding that ISP makes mocking easier in unit tests and reduces the surface area of changes.

**🚩 Red Signal:** Creates one giant interface per domain entity with 20+ methods that every consumer must implement or mock.

---

### 6. How do you apply SOLID principles when designing a background job processing system?

- **SRP:** Separate job scheduling, execution, retry logic, and dead-letter handling into distinct classes.
- **OCP:** New job types implement `IJob` or `IJobHandler<T>` — no changes to the processor/dispatcher.
- **DIP:** The dispatcher depends on `IJobHandler<T>`, resolved from DI; concrete handlers depend on injected repository/service abstractions.
- **LSP:** All `IJobHandler<T>` implementations honour the contract (return success/failure, do not throw unhandled exceptions).
- **ISP:** If some jobs need retry logic and others do not, separate `IRetryableJob` from `IJob`.

**Hint:** A great candidate draws parallels to libraries like MediatR, Hangfire, or MassTransit and explains how those frameworks already embody SOLID principles.

**🚩 Red Signal:** Puts all job logic in a single `BackgroundService` with a switch statement for each job type.

---

### 7. What is the captive dependency problem and how does it violate DIP?

A captive dependency occurs when a service with a shorter lifetime is injected into a service with a longer lifetime. For example, a `Scoped` `DbContext` injected into a `Singleton` service — the DbContext is captured and reused across all requests, causing threading issues and stale data.

**Fix:** Inject `IServiceScopeFactory` into the Singleton and create a scope per operation, or redesign the lifetimes.

**Hint:** ASP.NET Core 8+ throws by default (`ValidateScopes` is on in Development). The candidate should know this behaviour.

**🚩 Red Signal:** Registers everything as Singleton "for performance" without considering lifetime mismatches.

---

### 8. How do you avoid over-engineering when applying SOLID principles?

SOLID principles are guidelines, not laws. Over-engineering signs:
- Creating interfaces for classes that will never have a second implementation.
- Abstracting prematurely before understanding the domain.
- Adding layers of indirection that make the code harder to follow.

Pragmatic approach: apply SOLID where change is expected or where testability requires it. Use the **Rule of Three** — refactor to an abstraction when you see the same pattern for the third time.

**Hint:** A mature developer balances SOLID with YAGNI (You Ain't Gonna Need It) and KISS (Keep It Simple, Stupid). Look for nuanced thinking rather than dogmatic adherence.

**🚩 Red Signal:** Either applies every principle religiously to trivial code, or dismisses SOLID entirely as "academic nonsense."

---

### 9. How does the strategy pattern implement OCP in practice? Show a C# example.

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

### 10. When would you intentionally violate a SOLID principle and how do you justify it?

Examples of justified violations:
- **SRP:** A simple CRUD controller that handles validation, mapping, and persistence for a straightforward entity — the cost of abstraction outweighs the benefit.
- **OCP:** Modifying an existing class when adding an extension point would be over-engineering for a one-off change.
- **DIP:** Using a concrete class directly when it is a stable dependency unlikely to change (e.g., `List<T>`, `StringBuilder`).

**Hint:** The best candidates evaluate trade-offs: team size, project lifespan, rate of change, and complexity budget. They apply SOLID where it delivers value.

**🚩 Red Signal:** Either cannot imagine any scenario where violating SOLID is acceptable, or uses this as an excuse to never follow any principle.
