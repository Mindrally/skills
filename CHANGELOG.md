# Changelog

All notable changes to this library are recorded here. Releases are tagged on GitHub; watch the repo to get notified.

## [1.0.0] - 2026-10-08

First tagged release.

### Added
- `npx skills add Mindrally/skills` as the install method, with per-skill, bundle, and global examples
- Claude Code plugin marketplace manifest, so the library can be added with `/plugin marketplace add Mindrally/skills`
- Maintainer metadata in every skill's frontmatter
- Contributing guide, issue templates, and pull request template
- Skill bundles in the README for common stacks

### Changed
- 25 new skills and 8 updated skills synced from upstream Cursor rules (265 skills total)
- Five lowest-scoring skills rewritten after a `tessl skill review` pass: grpc-development, kafka-development, chrome-extension-development, analytics-data-analysis, design-systems

[1.0.0]: https://github.com/Mindrally/skills/releases/tag/v1.0.0
