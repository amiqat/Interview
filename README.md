# Senior .NET Developer — Interview Guide

> A structured, comprehensive interview question bank for evaluating senior .NET candidates.
> Each category is a separate file with difficulty levels, answer hints, code examples, follow-up questions, and red flags.

---

## Interview Flow

| Phase | Duration | Category | File |
|-------|----------|----------|------|
| 1 — Warm-up | ~10 min | SOLID Principles (S, O, D focus) | [solid-principles.md](solid-principles.md) |
| 2 — Core .NET | ~20 min | .NET Internals (ThreadPool, GC, async/await, ArrayPool, LOH) | [dotnet-internals.md](dotnet-internals.md) |
| 3 — Data Access | ~15 min | EF Core & LINQ | [ef-core-and-linq.md](ef-core-and-linq.md) |
| 4 — Database | ~15 min | SQL Server (Queries, Indexes, Bulk Ops, TVP) | [sql-server.md](sql-server.md) |
| 5 — API Design | ~15 min | REST API, Swagger & M2M Auth (JWT) | [rest-api.md](rest-api.md) |
| 6 — Architecture | ~15 min | Event-Driven Architecture (Concepts) | [event-driven.md](event-driven.md) |
| **Total** | **~90 min** | | |

> **Tip:** You don't need to ask every question. Pick 3-5 per category based on the candidate's experience. Start with 🟢 to build confidence, then go deeper with 🟡 and 🔴.

---

## Difficulty Levels

| Icon | Level | What it tests |
|------|-------|---------------|
| 🟢 | Easy | Fundamentals every senior dev should know cold |
| 🟡 | Medium | Deeper understanding, trade-offs, real-world experience |
| 🔴 | Hard | Expert-level, edge cases, architecture decisions |

---

## Scoring Rubric

| Score | Label | Description |
|-------|-------|-------------|
| 1 | ❌ No Hire | Cannot answer fundamentals, multiple red flags |
| 2 | ⚠️ Weak | Surface-level answers, misses trade-offs |
| 3 | ✅ Hire | Solid fundamentals, explains trade-offs, some depth |
| 4 | 🌟 Strong Hire | Deep expertise, real-world examples, mentorship potential |

---

## How to Use This Guide

1. **Before the interview** — Review the category files and pre-select questions based on the role requirements
2. **During the interview** — Use the hints to evaluate answers; watch for 🚩 red signals
3. **Follow-ups** — Each question has follow-up prompts to dig deeper when answers are surface-level
4. **Scoring** — Rate each category 1-4 using the rubric above, then make an overall decision
5. **Scenarios** — Each file includes scenario-based questions (marked with 💡) to test real-world problem solving

---

## Question Format

```
### 🟢/🟡/🔴 N. Question title

**Hint:** Short answer keywords and key concepts.

**Follow-up:** Deeper probe questions.

**🚩 Red Signal:** What a weak or concerning answer looks like.
```

---

## Categories

| # | Category | Questions | Topics |
|---|----------|-----------|--------|
| 1 | [SOLID Principles](solid-principles.md) | 15 | SRP, OCP, DIP, ISP, LSP, DI container, real-world refactoring |
| 2 | [.NET Internals](dotnet-internals.md) | 22 | ThreadPool, GC generations, Server/Workstation GC, LOH, ArrayPool, async/await state machine, ValueTask, Channels |
| 3 | [EF Core & LINQ](ef-core-and-linq.md) | 18 | Change tracking, N+1, compiled queries, raw SQL, interceptors, IQueryable vs IEnumerable, expression trees |
| 4 | [SQL Server](sql-server.md) | 20 | CTEs, window functions, execution plans, indexes, SqlBulkCopy, MERGE, temp tables, TVP, parameter sniffing, deadlocks |
| 5 | [REST API & Auth](rest-api.md) | 18 | HTTP conventions, versioning, HATEOAS, Swagger, minimal APIs, JWT, OAuth 2.0 M2M, rate limiting, middleware |
| 6 | [Event-Driven](event-driven.md) | 16 | Pub/sub, delivery guarantees, idempotency, outbox, sagas, event sourcing, CQRS, DLQ, backpressure, observability |