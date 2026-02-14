# Core SOLID Principles

Fundamental questions on each of the five SOLID principles — SRP, OCP, DIP, LSP, and ISP.

---

### 1. 🟢 Your colleague's `UserService` class handles registration, sends emails, writes audit logs, and validates input. What principle does this violate and how would you fix it?

SRP states that a class should have only one reason to change — it should encapsulate one responsibility. A `UserService` that handles user registration, sends emails, writes audit logs, and validates input has four reasons to change.

**Fix:** Extract responsibilities into focused classes: `UserRegistrationService`, `IEmailSender`, `IAuditLogger`, `IUserValidator`. Each class has one axis of change.

**Hint:** Look for candidates who talk about *cohesion* — methods and data that belong together. SRP is not "one method per class"; it is about having a single reason to change.

**🚩 Red Signal:** Interprets SRP as "each class should have only one method" or cannot identify when a class has too many responsibilities.

---

### 2. 🟡 Your e-commerce system needs a new payment method. The current `ProcessPayment` method has a growing switch statement for each provider. How should this be redesigned?

OCP states that software entities should be open for extension but closed for modification. Instead of adding a new `if`/`switch` branch to a `ProcessPayment` method for every new provider, define an `IPaymentProcessor` interface and create a new implementation (`StripePaymentProcessor`, `PayPalPaymentProcessor`). Register them in the DI container and resolve via a factory or strategy pattern.

**Hint:** Great answers mention that the DI container and the strategy/factory pattern are the practical mechanisms for achieving OCP in .NET. Also look for awareness that OCP does not mean *never* modifying existing code — it means designing extension points where change is expected.

**🚩 Red Signal:** Adds every new feature by modifying a giant `switch` statement, or claims you should never modify any existing code under any circumstance.

---

### 3. 🟡 Your ASP.NET Core controller directly instantiates `SqlOrderRepository`. How does this violate DIP and how would you fix it using ASP.NET Core's built-in DI?

DIP states: (A) High-level modules should not depend on low-level modules — both should depend on abstractions. (B) Abstractions should not depend on details — details should depend on abstractions.

In ASP.NET Core, this is realised through the built-in DI container: controllers (high-level) depend on interfaces (`IOrderRepository`), and the DI container wires in concrete implementations (`SqlOrderRepository`). The controller never knows about SQL Server.

**Hint:** A strong candidate distinguishes DIP (the principle) from DI (the pattern) from IoC containers (the tool). They should also explain service lifetimes: `Transient`, `Scoped`, `Singleton` — and the captive dependency problem (a Scoped service injected into a Singleton).

**🚩 Red Signal:** Equates DIP with "using an IoC container" without understanding why abstractions matter, or creates interfaces for every class without purpose (over-abstraction).

---

### 4. 🟡 A `Square` class inherits from `Rectangle` and overrides the `Width` setter to also set `Height`. Code that resizes rectangles now behaves unexpectedly. What principle is being violated and why?

LSP states that objects of a derived class should be substitutable for objects of the base class without altering the correctness of the program. The classic violation: `Square` inherits from `Rectangle` and overrides `Width`/`Height` setters to keep them equal — code that expects independent width/height breaks.

In .NET: a `ReadOnlyCollection<T>` that inherits from a mutable `IList<T>` and throws `NotSupportedException` on `Add()` violates LSP because callers of `IList<T>` expect `Add` to work.

**Hint:** LSP is about *behavioural* compatibility, not just method signatures. Look for candidates who can discuss preconditions, postconditions, and invariants.

**🚩 Red Signal:** Has never heard of LSP or thinks it is only about method signature compatibility.

---

### 5. 🟢 Your team's `IUserService` has 20 methods. A new consumer only needs `Login`. Mocking in tests requires implementing all 20 methods. What principle is being violated and how do you fix it?

ISP states that clients should not be forced to depend on interfaces they do not use. A fat `IUserService` with `Register`, `Login`, `UpdateProfile`, `DeleteAccount`, `GenerateReport`, and 15 other methods forces every consumer and test mock to implement methods they don't need. Split into focused interfaces: `IUserAuthentication`, `IUserRegistration`, `IUserProfileManager`.

**Hint:** In practice, ISP aligns with SRP at the interface level. Look for understanding that ISP makes mocking easier in unit tests and reduces the surface area of changes.

**🚩 Red Signal:** Creates one giant interface per domain entity with 20+ methods that every consumer must implement or mock.
