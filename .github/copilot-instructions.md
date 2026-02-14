# Copilot Instructions

## Project Overview

This is a **content-only Markdown repo** — a structured interview question bank for evaluating mid-to-senior .NET developers. There is no source code, no build system, and no tests. All output is Markdown files organized into category folders.

## Question Format (strict)

Every question **must** follow the template in [TEMPLATE.md](../TEMPLATE.md):

```markdown
### [number]. [🟢|🟡|🔴] [Question text]

[Model answer — 3-8 sentences. Code snippets if relevant.]

**Hint:** [What interviewers should look for; follow-up probes.]

**🚩 Red Signal:** [Specific bad-answer examples indicating a fundamental gap.]
```

- Difficulty emoji is **part of the heading**: `### 5. 🟡 How does…`
- Questions within a file are separated by `---` (horizontal rule)
- Numbering is sequential **per file**, starting at 1
- Question types: ⚡ Quick/Direct, 💬 Conceptual, 🎯 Scenario — aim for a mix

## Difficulty Levels

| Emoji | Level  | Criteria                                                             |
| ----- | ------ | -------------------------------------------------------------------- |
| 🟢    | Easy   | Foundational — expected of any mid/senior developer                  |
| 🟡    | Medium | Requires practical experience and deeper understanding               |
| 🔴    | Hard   | Advanced — only strong senior/principal-level candidates answer well |

## File & Folder Conventions

- Each category lives in its own folder (e.g., `dotnet/`, `sql-server/`, `event-driven/`)
- Every folder has a `README.md` with a category intro and a topics table linking to sub-files
- Sub-files use lowercase-with-hyphens: `query-performance.md`, `bulk-operations.md`
- Cross-cutting scenario questions go in `real-world-scenarios/`; domain-specific scenarios stay in their category folder
- The root `README.md` is the master index — update its categories table and question count when adding/removing questions

## Writing Style

- Questions must be open-ended ("how", "why", "walk me through") — never trivia
- Model answers are interviewer guides, not scripts — concise (3-8 sentences)
- Hints target principal-level interviewers: include follow-up probes and what distinguishes good from great
- Red Signals describe **specific** dangerous beliefs or practices, not vague "doesn't know X"
- Use fenced code blocks with language tags (`csharp`, `sql`) for code examples; keep snippets short
- Avoid jargon without explanation

## When Adding or Modifying Content

1. Place the question in the correct category folder and sub-file
2. Number it sequentially within that file
3. Include all four parts: heading with difficulty, model answer, Hint, 🚩 Red Signal
4. Separate from adjacent questions with `---`
5. Update the folder's `README.md` topics table (question count, difficulty dots)
6. Update the root `README.md` categories table (question count per category, total)
7. Refer to `EVALUATION.md` for the 1-4 scoring rubric context; questions should be assessable on that scale
