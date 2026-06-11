# Contributing to Frontend Engineering Lab

Thank you for your interest in contributing!

This guide explains how to contribute and the structure every contribution should follow. Following a consistent format keeps the knowledge base high-quality, searchable, and easy to read.

## Types of Contributions

- **Technical articles** => patterns, techniques, deep dives
- **Architecture discussions** => design decisions and their tradeoffs
- **Performance optimization techniques** => with measurements where possible
- **Code examples** => concise, production-oriented snippets
- **Case studies** =>real incidents, migrations, refactors, lessons learned
- **Best practices** => with the reasoning behind them
- **Interview preparation resources** => grounded in real engineering
- **Improvements** => fixing typos, clarifying explanations, updating outdated content

## Contribution Principles

- Focus on **practical engineering knowledge**, things you'd tell a teammate
- **Explain tradeoffs**, almost nothing in engineering is absolute
- Keep examples **concise and production-oriented**
- **Avoid promotional content**, no product pitches, affiliate links, or self-promotion
- Write for clarity, assume the reader is smart but unfamiliar with the topic
- **Original content only**, don't copy articles from elsewhere. Linking to references is encouraged

## Article Structure

Articles follow a common structure so the knowledge base stays consistent and easy to read. Use the sections that apply to your article and add any others it needs, only the metadata header is required. Start by copying a template from [`templates/`](templates/):

- [`templates/article-template.md`](templates/article-template.md) => for patterns, techniques, and deep dives
- [`templates/case-study-template.md`](templates/case-study-template.md) => for real-world stories and incidents

### 1. Metadata header (required)

Every article starts with this header:

```markdown
# Title of Your Article

> **Topic:** React · **Level:** Intermediate · **Author:** [@yourhandle](https://github.com/yourhandle)
```

- **Topic** => the section the article belongs to (React, TypeScript, Performance, etc.)
- **Level** => `Beginner`, `Intermediate`, or `Advanced`
- **Author** => your GitHub handle (so you get credit!)

### 2. Recommended sections

Most articles work well with this shape, use what applies:

| Section | Purpose |
|---|---|
| **The Problem** | What problem does this solve? Why should the reader care? |
| **The Solution / The Pattern** | Your main content, with code examples where relevant |
| **Tradeoffs** | When does this approach shine? When does it hurt? |
| **Key Takeaways** | 3–5 bullet points the reader should remember |

Other sections you can add: **Background/Context**, **Alternatives Considered**, **References**. Whatever sections you choose, try to cover the problem, the approach, and its tradeoffs somewhere in the article.

### 3. Code examples

- Use fenced code blocks with a language tag (` ```tsx `, ` ```ts `, ` ```css `)
- Keep snippets focused, show the relevant part, not the whole app
- Prefer TypeScript for JavaScript examples
- Show **before/after** when demonstrating an improvement

## File Naming & Placement

- Place your article in the most relevant top-level folder (e.g. `react/`, `performance/`)
- Use **kebab-case** file names that describe the content:
  - ✅ `react/error-boundaries-in-practice.md`
  - ✅ `performance/reducing-bundle-size-with-code-splitting.md`
  - ❌ `react/MyArticle.md`, `performance/tips.md`
- One article per file
- Add a link to your article in the folder's `README.md` index

## Proposing a New Top-Level Topic

The existing folders are a starting point, not a limit. If you want to write about an area we haven't declared yet (e.g. `animations/`, `security/`, `build-tooling/`, `developer-experience/`, `css/`), you're welcome to propose it.

To add a new top-level topic:

1. **Open a content proposal issue first** describing the topic and why it deserves its own section. This avoids overlap with existing folders (e.g. a CSS performance article likely belongs in `performance/`, not a new folder)
2. Once agreed, create the folder using a **kebab-case** name (e.g. `build-tooling/`)
3. Add a `README.md` to the folder that follows the same pattern as the existing sections:
   - A one-line description of the topic
   - An **Articles** index section (where contributions get linked)
   - An **Ideas for Contributions** list with a few starter ideas
   - A link back to this guide and the [article template](templates/article-template.md)
4. Add the new section to the **What You'll Find Here** table and the **Repository Structure** tree in the root `README.md`
5. Include your first article in the same PR, a new section should never be empty

Articles inside the new topic follow the same pattern as everywhere else: the required metadata header, plus the recommended sections that apply (The Problem, The Solution, Tradeoffs, Key Takeaways).

## Pull Request Process

1. **Fork** the repository
2. **Create a feature branch** with a descriptive name:
   - `add/react-error-boundaries`
   - `fix/typo-in-testing-guide`
3. **Commit your changes** with a clear message:
   - `Add article on error boundaries in React`
4. **Submit a Pull Request** using the PR template
5. A maintainer will review your contribution. Reviews focus on:
   - Accuracy and clarity
   - Following the article structure
   - Practical, non-promotional content

We may suggest edits, this is normal and collaborative, not a rejection. Most PRs are merged after a round of light feedback.

### Proposing an idea first

Not sure if your topic fits? Open a **content proposal issue** before writing. It saves you time and lets the community weigh in.

## Writing Tips

- **Lead with the problem**, not the tool
- Short paragraphs beat walls of text
- Use headings, lists, and tables to make content scannable
- A small, real example beats a long, abstract explanation
- It's okay to say *"it depends"*, just explain what it depends on
- English doesn't need to be perfect. Clarity matters more than grammar, reviewers will help polish

## Code of Conduct

By participating, you agree to follow our [Code of Conduct](CODE_OF_CONDUCT.md). Be kind, be constructive, and remember everyone here is learning.

## Questions?

Open an issue with the `question` label, we're happy to help you get your first contribution merged. 
