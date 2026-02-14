# Senior .NET Developer Interview Questions

A structured question bank for evaluating senior .NET developer candidates at a principal-interviewer level. Questions are organised into category folders, each with its own README containing 10 questions with interviewer hints and red-signal indicators.

## Quick Links

- 📋 [Question Template](TEMPLATE.md) — How to write new questions
- 📊 [Evaluation Guide](EVALUATION.md) — Scoring rubric, scorecard, and interview process

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
| 9 | Scenario-Based | [scenario-based/](scenario-based/) | Production debugging, migration, resilience, incident response | 10 |

**Total: 90 questions across 9 categories**

## Question Format

Each question follows a consistent structure:

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
│   └── README.md              # SRP, OCP, DIP, LSP, ISP
├── event-driven/
│   └── README.md              # Events, Sagas, Outbox, idempotency
├── architecture-patterns/
│   └── README.md              # Clean Architecture, CQRS, design patterns
└── scenario-based/
    └── README.md              # Real-world scenario questions
```