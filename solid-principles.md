# SOLID Principles — Senior Interview Questions (Focus: S, O, D)

## Single Responsibility Principle (SRP)

### 1. What is the Single Responsibility Principle?

**Hint:** A class should have one, and only one, reason to change. "Reason to change" = one actor/stakeholder. Separate concerns: don't mix data access, business logic, and presentation in one class.

**🚩 Red Signal:** Defines SRP as "a class should do one thing" without mentioning "reason to change" or stakeholders.

---

### 2. How do you identify SRP violations in existing code?

**Hint:** Look for classes with many dependencies (constructor with 10+ parameters), methods that serve different use cases, mixed abstraction levels, or class names with "And" / "Manager" / "Service" that do too many things. Large files are a smell, not proof.

**🚩 Red Signal:** Cannot give concrete examples of SRP violations, or thinks SRP means one method per class.

---

### 3. Give a practical example of refactoring an SRP violation.

**Hint:** A `UserService` that handles registration, email sending, and PDF generation → split into `UserRegistrationService`, `EmailNotificationService`, and `ReportGenerator`. Each has a single reason to change and a clear owner.

**🚩 Red Signal:** Over-applies SRP, creating dozens of trivial classes (one method each) that add complexity without clarity.

---

## Open/Closed Principle (OCP)

### 4. What is the Open/Closed Principle?

**Hint:** Software entities should be open for extension, closed for modification. Add new behavior by adding new code (new classes, new implementations) rather than modifying existing, tested code. Achieved through abstractions, interfaces, and polymorphism.

**🚩 Red Signal:** Cannot explain how to extend behavior without modifying the class, or thinks OCP means never changing any code.

---

### 5. How do you apply OCP in practice?

**Hint:** Use interfaces/abstract classes for extension points. Strategy pattern: inject different implementations. Plugin architecture. In .NET: `IPaymentProcessor` interface with `CreditCardProcessor`, `PayPalProcessor` — adding `CryptoProcessor` doesn't change existing code.

**🚩 Red Signal:** Adds `if/else` or `switch` for every new type instead of leveraging polymorphism.

---

### 6. When is OCP not worth the added abstraction?

**Hint:** When requirements are stable and unlikely to change. Over-engineering simple logic with excessive abstractions adds complexity. Apply OCP at known variation points — don't abstract everything "just in case." YAGNI (You Ain't Gonna Need It) still applies.

**🚩 Red Signal:** Applies OCP everywhere blindly, creating layers of abstraction for code that will never change.

---

## Dependency Inversion Principle (DIP)

### 7. What is the Dependency Inversion Principle?

**Hint:** High-level modules should not depend on low-level modules. Both should depend on abstractions. Abstractions should not depend on details — details should depend on abstractions. Typically: depend on `IRepository`, not `SqlRepository`.

**🚩 Red Signal:** Confuses DIP with Dependency Injection (DI). DIP is the principle; DI is one technique to achieve it.

---

### 8. How does Dependency Injection in .NET support DIP?

**Hint:** The built-in DI container (`IServiceCollection`) registers abstractions mapped to implementations. Constructor injection provides dependencies. Lifetimes: `Transient`, `Scoped`, `Singleton`. High-level code depends on interfaces; the container wires in the concrete types.

**🚩 Red Signal:** Manually instantiates dependencies inside classes (`new SqlRepository()`) or doesn't understand DI lifetimes (e.g., injecting `Scoped` into `Singleton`).

---

### 9. What is the Captive Dependency problem?

**Hint:** A `Scoped` or `Transient` service is injected into a `Singleton` — the shorter-lived service is "captured" and kept alive for the app's lifetime. Causes stale data, connection leaks, or thread-safety issues. In .NET, enable `ValidateScopes` in development to catch this.

**🚩 Red Signal:** Doesn't know what captive dependency is, or never enables scope validation during development.

---

### 10. How do you test code that follows DIP?

**Hint:** Inject mock/stub implementations of interfaces in unit tests. Use libraries like `Moq` or `NSubstitute`. Because high-level code depends on abstractions, you can replace any dependency without changing the class under test. This is the testability benefit of DIP.

**🚩 Red Signal:** Tests require a real database or external service because dependencies are hard-coded, not injected.

---

## Interface Segregation & Liskov (Brief)

### 11. Can you briefly explain ISP and LSP?

**Hint:** ISP: Clients should not be forced to depend on interfaces they don't use — prefer small, focused interfaces over large "fat" ones. LSP: Subtypes must be substitutable for their base types without altering correctness — e.g., a `ReadOnlyRepository` should not throw `NotImplementedException` on `Save()`.

**🚩 Red Signal:** Creates large interfaces with 20+ methods, or violates LSP by throwing `NotImplementedException` in derived classes.

---

### 12. How do SOLID principles work together in a real .NET project?

**Hint:** SRP keeps classes focused. OCP makes them extensible. DIP makes them testable and loosely coupled. ISP keeps interfaces lean. LSP ensures polymorphism works correctly. Together: small, focused classes behind interfaces, composed via DI, easily testable and extensible.

**🚩 Red Signal:** Treats SOLID as academic theory with no practical application, or applies each principle in isolation without seeing how they complement each other.
