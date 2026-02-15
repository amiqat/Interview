# Evaluation Guide

A structured framework for principal-level interviewers to assess senior .NET developer candidates consistently and fairly.

---

## Scoring Rubric

Rate each category on a 1–4 scale:

| Score | Level | Description |
|-------|-------|-------------|
| **4** | **Expert** | Deep, nuanced understanding. Explains trade-offs unprompted. Gives real-world examples from experience. Could teach the topic. |
| **3** | **Strong** | Solid understanding of concepts and practical application. Needs minor prompting for edge cases. Production-ready knowledge. |
| **2** | **Basic** | Knows the basics but lacks depth. Cannot discuss trade-offs or has only textbook knowledge. Needs mentoring in this area. |
| **1** | **Gap** | Significant misunderstanding or no knowledge. Red signals present. Risky to hire for a senior role without this competency. |

---

## Scorecard

Copy and fill this scorecard for each candidate:

```
Candidate: _______________
Date:      _______________
Interviewer: _______________

| Category                  | Score (1-4) | Notes |
|---------------------------|-------------|-------|
| .NET Internals            |             |       |
| .NET Modern (8/10)        |             |       |
| EF Core & LINQ            |             |       |
| SQL Server                |             |       |
| REST API & Auth           |             |       |
| SOLID Principles          |             |       |
| Event-Driven              |             |       |
| Architecture & Patterns   |             |       |
| Real-World Scenarios      |             |       |
| Communication & Clarity   |             |       |

Overall: ___ / 40
Recommendation: [ ] Strong Hire  [ ] Hire  [ ] No Hire  [ ] Strong No Hire
```

> **Note:** The scorecard has 10 categories instead of 8 question folders. The .NET category is split into two rows (.NET Internals and .NET Modern) to reflect its two distinct topic areas, and Communication & Clarity is a cross-cutting soft-skill assessment.

---

## How to Conduct the Interview

### Before the Interview
1. **Select 3–4 categories** most relevant to the role.
2. **Pick 2–3 questions per category** — mix foundational and deep questions.
3. **Prepare 1–2 real-world scenario questions** — these reveal how the candidate thinks under realistic conditions (see [real-world-scenarios/](real-world-scenarios/)).
4. **Review the Hints** — know what a strong answer looks like so you can assess in real time.

### During the Interview (60–90 minutes)

| Phase | Time | Activity |
|-------|------|----------|
| Intro | 5 min | Introduce yourself, explain the format, put the candidate at ease |
| Technical | 40–60 min | Work through selected questions across categories |
| Scenarios | 15–20 min | Present 1–2 scenario-based questions |
| Candidate Q&A | 10 min | Let the candidate ask questions about the team/role |

### Questioning Technique
- **Start broad, then drill down.** Begin with "Explain X" then follow up with "What happens if Y?" or "How would you handle Z in production?"
- **Follow the Hints.** Each question includes what to listen for — use these to decide when to probe deeper.
- **Watch for Red Signals.** These are not just "wrong answers" — they indicate gaps that are expensive to fix after hiring.
- **Give the candidate space.** If they're heading in the right direction, let them finish. Interrupting breaks their flow and hides their real depth.
- **Ask "why" more than "what."** "What is the Thread Pool?" is a factual recall. "Why does the Thread Pool use a hill-climbing algorithm instead of creating threads immediately?" reveals understanding.

### After the Interview
1. **Score each category immediately** — delay causes bias.
2. **Record specific examples** — "Explained the Outbox Pattern with a real production example" is better than "good at event-driven."
3. **Note Red Signals explicitly** — these are hiring risks and should be discussed in the debrief.
4. **Make a recommendation** — Use the scorecard. Be honest and specific.

---

## What to Look For at the Senior Level

A senior .NET developer should demonstrate:

- **Depth in core areas** — Not just "I've used EF Core" but "I've diagnosed and fixed N+1 problems in production."
- **Trade-off thinking** — Every pattern, technology, and approach has costs. Senior developers articulate these without being asked.
- **Production experience** — They've debugged thread starvation, handled migration failures, tuned SQL queries. Textbook knowledge is insufficient.
- **Pragmatism** — They know when to apply patterns and when the pattern adds more complexity than it solves.
- **Communication** — They explain complex topics clearly. This matters for code reviews, mentoring, and architecture discussions.

---

## Red Signal Summary

These are the most critical red signals across all categories. Any one of these should trigger a serious discussion about the hire decision:

| Red Signal | Why It Matters |
|---|---|
| Blocks thread pool threads with `Task.Result` in async code | Will cause production outages under load |
| No awareness of N+1 query problems | Will write code that brings the database down |
| Concatenates user input into SQL strings | SQL injection vulnerability |
| Cannot explain async/await beyond "it makes things faster" | Will write broken async code that's hard to debug |
| No strategy for handling partial failures in distributed systems | Systems will fail silently and corrupt data |
| Registers all DI services as Singleton | Will cause threading bugs and stale data |
| Never looks at SQL execution plans | Will write queries that work in dev but fail at production scale |
| Dismisses testing as "QA's job" | Will produce unreliable code and resist best practices |
