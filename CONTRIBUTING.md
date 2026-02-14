# Contributing

Thank you for helping improve this interview question bank! Here's how to contribute effectively.

## Adding Questions

1. **Use the template** — Follow the format in [TEMPLATE.md](TEMPLATE.md).
2. **Include a difficulty level** — Every question must have one of:
   - 🟢 **Easy** — Foundational knowledge expected of any senior developer.
   - 🟡 **Medium** — Requires practical experience and deeper understanding.
   - 🔴 **Hard** — Advanced topic; only strong senior/principal-level candidates will answer well.
3. **Place in the right category** — Add your question to the appropriate folder's `README.md`.
4. **Scenario-based questions belong in their category** — If a scenario is specific to SQL Server, add it to `sql-server/`. Cross-cutting scenarios go in `real-world-scenarios/`.

## Question Quality Checklist

Before submitting a PR, verify your question meets these criteria:

- [ ] Open-ended (asks "how", "why", "walk me through" — not trivia)
- [ ] Has a difficulty level (🟢, 🟡, or 🔴)
- [ ] Includes a **Hint** with what to look for and follow-up probes
- [ ] Includes a **🚩 Red Signal** with specific bad-answer examples
- [ ] Uses `---` separator between questions
- [ ] Does not duplicate an existing question

## File Naming

- Each category folder contains a `README.md` as the main question set.
- Additional files use lowercase with hyphens: `advanced-topics.md`, `deep-dives.md`.
- Never rename existing folders without updating the root `README.md` index.

## Style Guidelines

- Write in clear, professional English.
- Keep model answers concise (3–8 sentences).
- Use fenced code blocks with language tags for code examples.
- Avoid jargon without explanation — the interviewer may not be a specialist in every topic.

## Pull Request Process

1. Fork the repository and create a feature branch.
2. Add or update questions following the template.
3. Ensure all questions have difficulty levels.
4. Update the root `README.md` if adding a new category.
5. Open a PR with a clear description of what was added or changed.
