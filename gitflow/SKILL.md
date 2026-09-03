---
name: gitflow
description: "Branching, merging, versioning, and release conventions for the Gitflow workflow, covering main/develop branches, feature/release/hotfix branches, semantic versioning, and pull request rules. Use when creating a branch, naming a branch, planning a release or hotfix, writing a commit message, or setting up branch protection for a repository that follows Gitflow."
---

# Gitflow

This skill covers the Gitflow branching model: the two long-lived branches, the three supporting branch types, commit message conventions, semantic versioning, and the release and hotfix processes.

## Workflow for a Feature Change

1. **Sync develop** — Run `git checkout develop && git pull origin develop` so the feature branches from the latest development state.
2. **Create the feature branch** — Run `git checkout -b feature/123-user-authentication`, using the naming convention `feature/[issue-id]-descriptive-name`.
3. **Commit with conventional messages** — Use `type(scope): description` for every commit (see Commit Messages below).
4. **Rebase or merge develop in** — Before opening a PR, bring the branch up to date: `git fetch origin && git rebase origin/develop` (or merge, per team convention).
5. **Open a pull request into develop** — Require at least one approval and passing CI checks; never commit directly to `develop`.
6. **Merge and clean up** — After merge, delete the feature branch both locally and on the remote: `git branch -d feature/123-user-authentication && git push origin --delete feature/123-user-authentication`.

## Main Branches

### main (or master)

- Contains production-ready code only.
- Never commit directly to `main`.
- Only accepts merges from `hotfix/*` branches or `release/*` branches.
- Must be tagged with a version number after every merge (e.g. `git tag -a v1.2.0 -m "Release 1.2.0"`).

### develop

- The main integration branch; contains the latest delivered development changes.
- Source branch for all `feature/*` branches.
- Never commit directly to `develop` — all changes land through pull requests.

## Supporting Branches

### feature/*

- Branch from: `develop`.
- Merge back into: `develop`.
- Naming convention: `feature/[issue-id]-descriptive-name` (e.g. `feature/123-user-authentication`).
- Must be up to date with `develop` before opening a PR.
- Delete after merge.

### release/*

- Branch from: `develop`.
- Merge back into: `main` **and** `develop`.
- Naming convention: `release/vX.Y.Z` (e.g. `release/v1.2.0`).
- Scope is limited to bug fixes, documentation, and release-oriented tasks — no new features.
- Delete after merge.

### hotfix/*

- Branch from: `main`.
- Merge back into: `main` **and** `develop`.
- Naming convention: `hotfix/vX.Y.Z` (e.g. `hotfix/v1.2.1`).
- Reserved for urgent production fixes only.
- Delete after merge.

## Commit Messages

Use Conventional Commits format: `type(scope): description`.

| Type | Meaning |
|---|---|
| `feat` | New feature |
| `fix` | Bug fix |
| `docs` | Documentation changes |
| `style` | Formatting, missing semicolons, etc. — no logic change |
| `refactor` | Code restructuring without behavior change |
| `test` | Adding or correcting tests |
| `chore` | Maintenance tasks (build scripts, dependency bumps) |

Example:

```
feat(auth): add refresh-token rotation

fix(orders): prevent duplicate charge on retry

chore(deps): bump lodash to 4.17.21
```

## Semantic Versioning

Given a version `MAJOR.MINOR.PATCH`:

- **MAJOR** — incremented for incompatible/breaking API changes.
- **MINOR** — incremented for backwards-compatible new functionality.
- **PATCH** — incremented for backwards-compatible bug fixes.

## Pull Request Rules

1. All changes must go through pull requests — no exceptions for `main` or `develop`.
2. Minimum of one required approval before merge.
3. All CI checks must pass.
4. No direct commits to protected branches (`main`, `develop`).
5. The branch must be up to date with its target before merging.
6. Delete the branch after merge (both local and remote copies).

## Branch Protection Rules

Apply to both `main` and `develop`:

- Require pull request reviews before merging.
- Require status checks to pass before merging.
- Require branches to be up to date before merging.
- Include administrators in the restrictions (no bypassing rules with elevated permissions).
- No force pushes.
- No branch deletions.

## Release Process

1. Create a release branch from `develop`: `git checkout -b release/v1.2.0 develop`.
2. Bump version numbers in package manifests / changelogs.
3. Fix any release-specific issues (docs, last-minute bugs) — do not add new features.
4. Open a PR from the release branch into `main`.
5. After merge to `main`:
   - Tag the release: `git tag -a v1.2.0 -m "Release 1.2.0"` and push the tag.
   - Merge `main` back into `develop` so develop has the release commit and tag.
   - Delete the release branch.

## Hotfix Process

1. Create a hotfix branch from `main`: `git checkout -b hotfix/v1.2.1 main`.
2. Fix the production issue with the smallest possible change.
3. Bump the patch version.
4. Open a PR from the hotfix branch into `main`.
5. After merge to `main`:
   - Tag the release: `git tag -a v1.2.1 -m "Hotfix 1.2.1"` and push the tag.
   - Merge `main` back into `develop` so the fix isn't lost in the next release.
   - Delete the hotfix branch.
