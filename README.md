# Senior .NET Developer Interview Questions

A structured question bank for evaluating senior .NET developer candidates at a principal-interviewer level. Questions are organised into category folders, each with its own README containing 10 questions with difficulty levels, interviewer hints, and red-signal indicators.

## Quick Links

- 📋 [Question Template](TEMPLATE.md) — How to write new questions
- 📊 [Evaluation Guide](EVALUATION.md) — Scoring rubric, scorecard, and interview process
- 🤝 [Contributing](CONTRIBUTING.md) — How to add or improve questions
- 📄 [License](LICENSE) — MIT

## Difficulty Levels

| Emoji | Level | Meaning |
|-------|-------|---------|
| 🟢 | Easy | Foundational knowledge expected of any senior developer |
| 🟡 | Medium | Requires practical experience and deeper understanding |
| 🔴 | Hard | Advanced topic; only strong senior/principal-level candidates will answer well |

## Categories

| # | Category | Folder | Key Topics | Questions |
|---|----------|--------|------------|-----------|
| 1 | .NET Internals | [dotnet-internals/](dotnet-internals/) | Thread Pool, GC, async/await, ArrayPool, LOH, Span\<T\> | 10 |
| 2 | .NET Modern (8/10) | [dotnet-modern/](dotnet-modern/) | Minimal APIs vs controllers, Native AOT, endpoint filters, health checks | 10 |
| 3 | EF Core & LINQ | [ef-core-linq/](ef-core-linq/) | Query translation, change tracking, bulk operations, N+1, migrations | 10 |
| 4 | SQL Server | [sql-server/](sql-server/) | Indexes, complex queries, SqlBulkCopy, UPSERT, TVP, temp tables | 10 |
| 5 | REST API & Auth | [rest-api/](rest-api/) | REST conventions, Swagger/OpenAPI, JWT, M2M auth, CORS | 10 |
| 6 | SOLID Principles | [solid-principles/](solid-principles/) | SRP, OCP, DIP, LSP, ISP, practical trade-offs | 10 |
| 7 | Event-Driven Architecture | [event-driven/](event-driven/) | Events vs commands, eventual consistency, Outbox, Sagas, idempotency | 10 |
| 8 | Architecture & Patterns | [architecture-patterns/](architecture-patterns/) | Clean Architecture, CQRS, MediatR, Circuit Breaker, Options pattern | 10 |
| 9 | Real-World Scenarios | [real-world-scenarios/](real-world-scenarios/) | Production debugging, migration, resilience, incident response | 10 |

**Total: 90 questions across 9 categories**

## Question Format

Each question follows a consistent structure:

- **Difficulty** — 🟢 Easy, 🟡 Medium, or 🔴 Hard (shown in the question heading).
- **Question** — Open-ended interview question.
- **Hint** — What to look for in a strong answer. Follow-up probes for the interviewer.
- **🚩 Red Signal** — Specific warning signs that indicate a fundamental gap.

## How to Use

1. **Before the interview:** Pick 3–4 categories relevant to the role. Select 2–3 questions per category.
2. **During the interview:** Start with foundational questions, then drill deeper. Use **Hints** to guide follow-ups.
3. **Watch for 🚩 Red Signals:** These indicate gaps that are expensive to fix after hiring.
4. **After the interview:** Use the [Evaluation Guide](EVALUATION.md) scorecard to record scores and make a recommendation.

See the [Evaluation Guide](EVALUATION.md) for the full interview process, scoring rubric, and scorecard template.

## Folder Structure

```
├── README.md                  # This file — index and overview
├── TEMPLATE.md                # Question template for contributors
├── EVALUATION.md              # Scoring rubric and interview guide
├── CONTRIBUTING.md            # How to add or improve questions
├── LICENSE                    # MIT license
├── dotnet-internals/
│   └── README.md              # Thread Pool, GC, async/await, ArrayPool, LOH
├── dotnet-modern/
│   └── README.md              # .NET 8/10, minimal APIs, Native AOT
├── ef-core-linq/
│   └── README.md              # EF Core, LINQ, change tracking, bulk ops
├── sql-server/
│   └── README.md              # Indexes, queries, bulk copy, TVP
├── rest-api/
│   └── README.md              # REST conventions, Swagger, JWT, M2M auth
├── solid-principles/
│   ├── README.md              # Intro + index
│   ├── core-principles.md     # SRP, OCP, DIP, LSP, ISP
│   └── applied-solid.md       # Trade-offs, patterns, when to break the rules
├── event-driven/
│   ├── README.md              # Intro + index
│   ├── fundamentals.md        # EDA concepts, events vs commands, DLQ
│   └── patterns.md            # Outbox, Saga, idempotency, broker vs streaming
├── architecture-patterns/
│   ├── README.md              # Intro + index
│   ├── architecture.md        # Clean Architecture, vertical slices, CQRS
│   └── design-patterns.md     # Mediator, Options, Circuit Breaker, Decorator
└── real-world-scenarios/
    ├── README.md              # Intro + index
    ├── debugging-performance.md # Production diagnosis, EF Core performance
    ├── system-design.md       # Multi-tenant, real-time, high-throughput
    └── engineering-practices.md # Migration, resilience, CI/CD, testing
```