# .NET Developer Interview Questions

A structured question bank for evaluating mid-to-senior .NET developer candidates. Questions range from quick conversation openers to deep scenario-based problems, organised into category folders with difficulty levels, interviewer hints, and red-signal indicators.

## Quick Links

- 📋 [Question Template](TEMPLATE.md) — How to write new questions
- 📊 [Evaluation Guide](EVALUATION.md) — Scoring rubric, scorecard, and interview process
- 🤝 [Contributing](CONTRIBUTING.md) — How to add or improve questions
- 📄 [License](LICENSE) — MIT

## Difficulty Levels

| Emoji | Level | Meaning |
|-------|-------|---------|
| 🟢 | Easy | Foundational — expected of any mid/senior developer |
| 🟡 | Medium | Requires practical experience and deeper understanding |
| 🔴 | Hard | Advanced — only strong senior/principal-level candidates answer well |

## Question Types

Each category contains a **balanced mix** of question types:

| Type | Purpose | Example |
|------|---------|---------|
| ⚡ Quick / Direct | Open conversation, gauge baseline | "What does HTTP 429 mean?" |
| 💬 Conceptual | Explore understanding and reasoning | "How does the GC's generational model work?" |
| 🎯 Scenario | Test problem-solving with real situations | "Your API memory grows over time — what do you investigate?" |

Start with quick questions, use conceptual questions to probe depth, and save scenario questions for the strongest signal.

## Categories

| # | Category | Folder | Key Topics | Questions |
|---|----------|--------|------------|-----------|
| 1 | .NET | [dotnet/](dotnet/) | Thread Pool, GC, async/await, memory, Minimal APIs, .NET 8/10, DI | 22 |
| 2 | EF Core & LINQ | [ef-core-linq/](ef-core-linq/) | Query translation, change tracking, bulk operations, N+1, migrations | 13 |
| 3 | SQL Server | [sql-server/](sql-server/) | Indexes, queries, bulk ops, TVP, schema-based SQL questions | 18 |
| 4 | REST API & Auth | [rest-api/](rest-api/) | REST conventions, Swagger/OpenAPI, JWT, M2M auth, CORS | 15 |
| 5 | SOLID Principles | [solid-principles/](solid-principles/) | SRP, OCP, DIP, LSP, ISP, practical trade-offs | 10 |
| 6 | Event-Driven | [event-driven/](event-driven/) | Events vs commands, eventual consistency, Outbox, Sagas, idempotency | 13 |
| 7 | Architecture & Patterns | [architecture-patterns/](architecture-patterns/) | Clean Architecture, CQRS, MediatR, Circuit Breaker, Decorator | 12 |
| 8 | Real-World Scenarios | [real-world-scenarios/](real-world-scenarios/) | Production debugging, migration, resilience, incident response | 10 |

**Total: 113 questions across 8 categories**

## Question Format

Each question follows a consistent structure:

- **Difficulty** — 🟢 Easy, 🟡 Medium, or 🔴 Hard (in the heading)
- **Question** — Direct, conceptual, or scenario-based
- **Model answer** — Guide for the interviewer (not a script)
- **Hint** — What to look for in a strong answer; follow-up probes
- **🚩 Red Signal** — Warning signs that indicate a fundamental gap

## How to Use

1. **Before the interview:** Pick 3–4 categories relevant to the role. Select a mix of quick, conceptual, and scenario questions.
2. **During the interview:** Start with 🟢 quick questions to warm up, then go deeper with 🟡 conceptual and 🔴 scenario questions.
3. **Watch for 🚩 Red Signals:** These indicate gaps that are expensive to fix after hiring.
4. **After the interview:** Use the [Evaluation Guide](EVALUATION.md) scorecard to record scores.

## Folder Structure

```
├── README.md                    # This file — index and overview
├── TEMPLATE.md                  # Question template for contributors
├── EVALUATION.md                # Scoring rubric and interview guide
├── CONTRIBUTING.md              # Contribution guidelines
├── LICENSE                      # MIT
├── dotnet/
│   ├── README.md                # .NET category intro + index
│   ├── internals.md             # Thread Pool, GC, async/await, memory
│   └── modern-dotnet.md         # Minimal APIs, .NET 8/10, DI, hosting
├── ef-core-linq/
│   ├── README.md                # EF Core & LINQ intro + index
│   ├── query-performance.md     # Query translation, tracking, N+1, compiled queries
│   └── data-management.md       # Bulk inserts, migrations, raw SQL, concurrency
├── sql-server/
│   ├── README.md                # SQL Server intro + index
│   ├── indexing-performance.md   # Clustered/non-clustered, covering, Query Store
│   ├── bulk-operations.md       # SqlBulkCopy, UPSERT, TVP, temp tables
│   ├── query-optimization.md    # Complex joins, SARGability, large data
│   └── schema-and-queries.md    # Schema-based SQL questions (write the query)
├── rest-api/
│   ├── README.md                # REST API & Auth intro + index
│   ├── api-design.md            # REST conventions, versioning, Swagger, errors
│   └── authentication.md        # JWT, M2M auth, CORS, authn vs authz
├── solid-principles/
│   ├── README.md                # SOLID intro + index
│   ├── core-principles.md       # SRP, OCP, DIP, LSP, ISP
│   └── applied-solid.md         # Trade-offs, patterns, when to break rules
├── event-driven/
│   ├── README.md                # Event-driven intro + index
│   ├── fundamentals.md          # EDA concepts, events vs commands, DLQ
│   └── patterns.md              # Outbox, Saga, idempotency, broker vs streaming
├── architecture-patterns/
│   ├── README.md                # Architecture intro + index
│   ├── architecture.md          # Clean Architecture, vertical slices, CQRS
│   └── design-patterns.md       # Mediator, Options, Circuit Breaker, Decorator
└── real-world-scenarios/
    ├── README.md                # Scenarios intro + index
    ├── debugging-performance.md # Production diagnosis, performance
    ├── system-design.md         # Multi-tenant, real-time, high-throughput
    └── engineering-practices.md # Migration, resilience, CI/CD, testing
```