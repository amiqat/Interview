# Architecture Patterns

Questions on software architecture patterns and architectural decision-making for .NET applications.

---

### 1. 🟡 A new project is set up with `Controllers/`, `Services/`, `Repositories/` folders. A developer adds EF Core references directly in the service layer. What architectural problems might this cause?

This is a classic **layered architecture** that's missing the key rule of **Clean Architecture**: the **Dependency Rule** — outer layers depend on inner layers, never the reverse.

Clean Architecture organises code into concentric layers with the **domain** at the centre:

- **Domain** — Entities, value objects, domain events, interfaces. No external dependencies.
- **Application** — Use cases, DTOs, validation. Depends only on Domain.
- **Infrastructure** — EF Core, external APIs, file system. Implements interfaces from Application/Domain.
- **Presentation** — Controllers, minimal API endpoints. Depends on Application.

Adding EF Core references in the service layer means the business logic is coupled to a specific persistence technology. This makes the service layer untestable without a database and impossible to swap the data access strategy.

**Hint:** Look for understanding that the key rule is the **Dependency Rule** — and that the candidate can discuss when Clean Architecture is overkill (small CRUD apps).

**🚩 Red Signal:** Cannot explain the dependency direction, or applies Clean Architecture to every project regardless of complexity.

---

### 2. 🔴 Your team's layered architecture means every feature change touches 5 files across 3 layers. Merge conflicts are constant and features take twice as long as expected. What alternative architecture would you propose?

**Layered (horizontal):** Organise by technical concern — Controllers, Services, Repositories. Changes to a feature touch multiple layers and files, leading to merge conflicts in shared folders.

**Vertical slice:** Organise by feature — each slice contains the handler, validation, data access, and response for a single use case. Libraries like MediatR enable this with `IRequest<T>` / `IRequestHandler<T, R>`.

**When to choose vertical slices:** Feature-heavy applications where each feature has distinct behaviour, teams that want to minimise merge conflicts, and codebases where layers have become overly generic.

**Hint:** A principal-level candidate doesn't dogmatically choose one — they explain the trade-offs and pick based on team size, project complexity, and rate of change.

**🚩 Red Signal:** Has only ever used one pattern and cannot articulate alternatives.

---

### 3. 🔴 Your read and write patterns are very different — complex validation on writes but simple flat reads. A single model serves both, and it's getting increasingly complex. How do you separate concerns?

CQRS (Command Query Responsibility Segregation) separates the **write model** (commands — optimised for consistency and validation) from the **read model** (queries — optimised for fast retrieval, possibly denormalised).

**Worth it when:** Read and write patterns are very different (exactly this scenario), high read:write ratio, or when combined with event sourcing.

**Not worth it when:** Simple CRUD with similar read/write shapes — CQRS adds unnecessary complexity.

**Hint:** A principal-level candidate should be able to explain CQRS without event sourcing (they are independent patterns) and discuss the consistency challenge of a separate read store.

**🚩 Red Signal:** Conflates CQRS with event sourcing, or implements CQRS for a basic CRUD application.

---

### 4. 🟡 A team member wraps every `DbSet<T>` in a generic `Repository<T>` that returns `IQueryable<T>`. They say it's "for testability." Do you agree? What are the problems?

The Repository pattern abstracts data access behind an interface (`IOrderRepository`), hiding the persistence mechanism from the business logic.

**Arguments for repositories:** Testability (mock the repository), swappable data stores, consistent query patterns.

**Arguments against with EF Core:** `DbContext` already implements Unit of Work + Repository (`DbSet<T>`). Adding a generic repository on top creates a "repository over repository" that limits EF Core features (LINQ composition, change tracking). Returning `IQueryable<T>` leaks the abstraction — callers can compose arbitrary queries that may not translate to SQL.

**Pragmatic approach:** Use repositories for complex aggregates with domain logic, skip them for simple queries, or use a thin "query service" instead.

**Hint:** A principal-level candidate has a nuanced opinion — they don't dismiss all patterns as unnecessary, nor do they blindly wrap everything.

**🚩 Red Signal:** Always wraps `DbContext` in a generic repository that returns `IQueryable<T>` (leaking the abstraction), or dismisses all patterns as unnecessary.
