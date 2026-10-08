# Contributing

Thanks for helping improve the Mindrally skills library. Contributions of every size are welcome: new skills, fixes to existing ones, better descriptions, or examples.

## Quick start

1. Fork the repo and create a branch: `git checkout -b add-<skill-name>`
2. Make your change (see below)
3. Install it locally to test: `npx skills add ./ --skill <skill-name>`
4. Open a pull request

## Adding a new skill

Create a new folder at the repo root using a kebab-case name, with a single `SKILL.md` inside:

```
my-new-skill/
└── SKILL.md
```

The file must start with YAML frontmatter:

```markdown
---
name: my-new-skill
description: One sentence saying what the skill covers and when to use it. Claude reads this to decide when to activate the skill.
metadata:
  maintainer: Mindrally
  source: https://github.com/Mindrally/skills
---

# My New Skill

Guidelines, conventions, and examples...
```

Guidelines:

- **`name`** must match the folder name exactly.
- **`description`** should be specific. "Expert in X" is weaker than "X development with Y and Z. Use when building A, B, or C." The description is what Claude matches against your request.
- **Be concrete.** Prefer code examples and explicit rules over general advice.
- **Keep it focused.** One technology or practice per skill. If a skill grows past about 500 lines, split it.
- **No secrets, no tracking, no network calls.** Skills are instructions, not programs.
- Add a row for the skill to the matching table in `README.md`.

## Improving an existing skill

Edit the `SKILL.md` directly. In your PR description, say what was wrong or missing and how the change helps. Before and after examples of Claude's output are the most convincing evidence.

## Requesting a skill

Open a [skill request](https://github.com/Mindrally/skills/issues/new?template=skill_request.yml). Requests that name the specific framework version and describe real use cases get picked up fastest.

## Style

- Plain markdown, no HTML.
- Fenced code blocks with a language tag.
- Headings in sentence case.
- No emojis in skill content.

## Review

Maintainers review PRs weekly. Small, single-skill PRs merge fastest. By contributing you agree that your contribution is licensed under the repo's Apache 2.0 license.
