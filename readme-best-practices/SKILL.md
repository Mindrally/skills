---
name: readme-best-practices
description: "Structure, tone, and content conventions for writing effective project README files, covering hooks, quick starts, feature presentation, badges, and common documentation sections. Use when writing a new README, rewriting an existing one, or reviewing README quality for a repository, library, or CLI tool."
---

# README Best Practices

This skill covers how to write a README that reads like a landing page rather than an API reference — the reader decides whether to keep reading within 3-5 seconds, so the first screen has to earn the rest.

## Workflow for Writing a README

1. **Draft the one-liner** — Write a bold, specific sentence stating what the project does and why someone should care. Avoid "A tool that..."; aim for a punchline.
2. **Write a working code example first** — Put a copy-pasteable example in the first 5-10 lines of content, before installation instructions. Show the value proposition immediately.
3. **Add badges** — Build status, version, license, and coverage badges directly under the title, if the project has CI/publishing set up.
4. **Write Quick Start** — Zero-to-running in under 30 seconds, with no placeholder values the reader has to mentally substitute.
5. **Fill in supporting sections** — Features, Usage, Configuration, Contributing, License — using the structure below, only including sections that carry real information.
6. **Verify every asset and link** — Confirm referenced images (screenshots, demo.gif) exist on disk and that internal links resolve before publishing.
7. **Read it cold** — Reread the first screen as if seeing the project for the first time; cut anything that doesn't help a decision to keep reading or stop.

## Opening Hook

- Start with a bold one-liner saying what the project does and why someone should care — not "A tool that...", a punchline.
- Put a working code example in the first 5 lines. Show the value proposition immediately, before explaining installation.
- Never open with "In today's fast-paced world..." or similar throat-clearing.
- Never close with "Happy coding!" or similar filler sign-offs.
- Avoid AI-marketing words: "seamless", "robust", "comprehensive", "cutting-edge", "powerful", "effortless". State what it does instead of how impressive it sounds.

## Standard Structure

A typical README benefits from these sections, roughly in this order — include only the ones that add real information for this project:

1. **Title + one-liner** — project name and the bold hook sentence.
2. **Badges** — build status, latest version, license, test coverage.
3. **Quick demo** — a code snippet, GIF, or screenshot showing the thing working.
4. **Features** — a two-column table, not a wall of bullets (see below).
5. **Installation** — the exact command(s) to install, per package manager if there's more than one.
6. **Quick Start / Usage** — copy-paste-ready minimal example, then a couple of more advanced examples.
7. **Configuration** — options, environment variables, config file format, with defaults noted.
8. **API Reference** (or a link to one) — for libraries with a non-trivial public surface.
9. **FAQ / Troubleshooting** — the 3-5 questions people actually ask in issues.
10. **Contributing** — how to set up the dev environment, run tests, and submit a PR; link to `CONTRIBUTING.md` if it exists.
11. **License** — name and link to the license file.
12. **Author / Acknowledgments** — credit maintainers and major dependencies.

## Feature Presentation

- Use feature tables (two columns: feature, description) instead of `**Feature:**` bullet lists — tables scan faster than repeated bold-prefix bullets.

```markdown
| Feature | Description |
|---|---|
| Zero-config | Works out of the box with sensible defaults |
| Streaming | Handles gigabyte-scale files without loading them into memory |
| Type-safe | Full TypeScript definitions, no `any` in the public API |
```

## Quick Start Requirements

- Must be copy-paste ready: zero to running in 30 seconds.
- Do not prefix shell commands with `$` — it breaks copy-paste.
- Show the install command and the minimal usage example together, not split across distant sections.

```bash
npm install awesome-lib

awesome-lib run --input data.csv --output report.json
```

```js
import { parse } from "awesome-lib";

const result = parse("data.csv");
console.log(result.summary);
```

## Prose and Formatting

- Vary sentence length and structure — mix one-liners with short paragraphs and tables. Walls of same-length bullets read as filler.
- Use headings to let readers jump straight to the section they need; don't force a linear read.
- Prefer runnable examples over prose descriptions of behavior wherever both are possible.
- Keep line-level formatting consistent: one fenced code block per language/command, explicit language tags (` ```bash `, ` ```json `) for syntax highlighting.

## Badges

- Use badges for objective, machine-checkable facts: CI status, published version, license, downloads, coverage.
- Keep the badge row short — 3-6 badges. A wall of badges is as noisy as a wall of bullets.
- Common sources: shields.io for custom badges, the CI provider's own badge markdown, npm/PyPI's official badge snippets.

## Assets and Links

- Check that referenced assets (`demo.gif`, screenshots) actually exist on disk before adding image links — a broken image in the first screen kills credibility instantly.
- Verify internal anchor links (table of contents, "see Configuration below") resolve to real headings.
- Prefer relative paths for repo-local assets so they render correctly on the git host and in package registries alike.

## Author / Contact Section

- Include a visual card or badge for the author/maintainer rather than plain text like "Made by username" — a GitHub profile badge, a small avatar + link, or a sponsor button reads as more intentional.
- For multi-maintainer projects, list maintainers with their role or area of ownership rather than a flat name list.

## Common Mistakes to Avoid

- Leading with installation instead of value — readers don't know why they should install it yet.
- Documenting every configuration option in prose instead of a table.
- Letting the README drift from the actual CLI/API surface — stale examples that no longer run are worse than no examples.
- Mixing marketing language ("blazing fast", "enterprise-grade") with technical documentation — pick one register and stay technical.
- Duplicating full API docs in the README when a generated reference (TypeDoc, Sphinx, godoc) already exists — link to it instead.
