# Question Template

Use this template when adding new interview questions. Copy the block below and fill in the fields.

---

```markdown
### [Number]. [🟢|🟡|🔴] [Question text — clear, specific, and open-ended]

[Concise model answer — 3-8 sentences covering the key points. Include code snippets if relevant.]

**Hint:** [What a strong answer includes. What follow-up questions to ask. What distinguishes a good answer from a great one.]

**🚩 Red Signal:** [Specific warning signs — incorrect beliefs, dangerous practices, or fundamental gaps that indicate the candidate is not at the required level.]
```

---

## Difficulty Levels

Every question must have a difficulty indicator in the heading:

| Emoji | Level | When to use |
|-------|-------|-------------|
| 🟢 | Easy | Foundational knowledge expected of any senior developer |
| 🟡 | Medium | Requires practical experience and deeper understanding |
| 🔴 | Hard | Advanced topic; only strong senior/principal-level candidates will answer well |

---

## Guidelines

1. **One concept per question** — Don't combine unrelated topics. If a question touches multiple areas, make it a scenario-based question.
2. **Open-ended, not trivia** — Avoid "what does X stand for?" questions. Ask "how", "why", "when", and "walk me through".
3. **Model answer is a guide, not a script** — Candidates may use different terminology or approaches. The Hint section helps you evaluate non-standard but valid answers.
4. **Hint is for the interviewer** — It should help a principal-level interviewer assess depth. Include follow-up probes.
5. **Red Signal is a deal-breaker** — These are not "less good" answers; they indicate a fundamental gap. Be specific about what the bad answer looks like.
6. **Include code when it adds clarity** — Use fenced code blocks with language tags. Keep snippets short and focused.
7. **Use `---` between questions** — Horizontal rules separate questions visually.

---

## Example

### 1. 🟡 How does the .NET Thread Pool decide when to add or remove threads?

The Thread Pool uses a hill-climbing algorithm: it adds a thread, measures throughput, and keeps going in the same direction if throughput improved. If throughput dropped, it reverses. This converges on the optimal thread count for the current workload.

The pool starts with `Environment.ProcessorCount` threads and injects new ones slowly (~1-2 per second) once all existing threads are busy.

**Hint:** A strong answer explains why creating threads on demand is expensive (~1 MB stack per OS thread), mentions `ThreadPool.SetMinThreads` for burst workloads, and discusses the relationship between I/O completion ports and the thread pool.

**🚩 Red Signal:** Suggests solving thread starvation by setting `SetMinThreads(1000, 1000)` without investigating why threads are being blocked in the first place.

---

## File Naming

- Place questions in the appropriate category folder.
- File name: `README.md` for the main question set in each folder.
- Additional question files: `advanced-topics.md`, `scenario-questions.md`, etc.
- Use lowercase with hyphens: `my-topic.md`.
