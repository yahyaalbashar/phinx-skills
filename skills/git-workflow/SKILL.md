---
name: git-workflow
description: "Senior-level Git workflow, branching strategy, commit conventions, and release management. Use this skill whenever creating branches, writing commits, managing PRs, resolving merge conflicts, setting up Git hooks, tagging releases, or discussing Git workflow. Also trigger when the user mentions Git, branching strategy, trunk-based development, GitFlow, conventional commits, rebasing, cherry-picking, release management, or changelogs."
---

# Git Workflow — Senior Engineer Standards

You are a senior engineer who uses Git as a communication tool, not just a backup system. Every commit tells a story. Every branch has a purpose.

## Branching Strategy

### Trunk-Based Development (Recommended for most teams)
- `main` is always deployable
- Short-lived feature branches (< 2 days ideally, never > 1 week)
- Feature flags for work-in-progress features
- Branch naming: `type/description` → `feat/user-auth`, `fix/payment-timeout`, `chore/update-deps`

### Branch prefixes
- `feat/` — New feature
- `fix/` — Bug fix
- `chore/` — Maintenance, deps, config
- `refactor/` — Code restructuring (no behavior change)
- `docs/` — Documentation only
- `test/` — Adding or fixing tests
- `hotfix/` — Emergency production fix

## Conventional Commits

### Format
```
type(scope): description

[optional body]

[optional footer]
```

### Examples
```
feat(auth): add JWT refresh token rotation
fix(api): handle null response from payment gateway
refactor(orders): extract discount calculation to service layer
chore(deps): upgrade React to v19
docs(api): add OpenAPI spec for user endpoints
test(auth): add integration tests for password reset flow
perf(queries): add composite index for order lookups

BREAKING CHANGE: auth endpoints now require API version header
```

### Rules
- Subject line: imperative mood, lowercase, no period, < 72 characters
- Body: explain WHAT and WHY, not HOW (code shows how)
- Footer: reference issues (`Closes #123`), note breaking changes
- One logical change per commit. Not "fix stuff" with 20 files.

## PR Best Practices

### PR should:
- Solve ONE problem or implement ONE feature
- Be reviewable in < 30 minutes (< 400 lines changed)
- Include a clear description: what, why, how to test
- Link to the ticket/issue
- Include screenshots/recordings for UI changes
- Pass all CI checks before requesting review

### PR description template
```markdown
## What
[Brief description of the change]

## Why
[Link to ticket. Business context.]

## How
[Technical approach. Key decisions made.]

## Testing
[How to test this. What was tested.]

## Screenshots
[If UI changes]
```

### Splitting large PRs
- Infrastructure/setup changes first (new table, new service scaffold)
- Core logic second (business rules, handlers)
- UI last (components, pages)
- Each PR should be independently deployable and not break main

## Git Operations

### Rebase vs Merge
- `git rebase main` for updating feature branches (clean linear history)
- `git merge --no-ff` for merging PRs to main (preserves branch context)
- NEVER force-push to shared branches
- Squash merge for feature branches with messy commit history

### Interactive rebase for cleanup
Before creating a PR, clean up your commits:
```bash
git rebase -i HEAD~5
# pick, squash, fixup, reword as needed
```
Goal: each commit in the PR should be a logical, reviewable unit.

### Useful Git commands
```bash
# Find who last changed a line
git blame -L 10,20 path/to/file

# Find which commit introduced a bug
git bisect start
git bisect bad HEAD
git bisect good v1.2.0

# See what changed in a file over time
git log --follow -p path/to/file

# Stash with a name
git stash push -m "WIP: payment flow"

# Cherry-pick a fix to release branch
git cherry-pick <commit-sha>
```

## Release Management

### Semantic Versioning (SemVer)
- `MAJOR.MINOR.PATCH` → `2.1.3`
- MAJOR: Breaking changes
- MINOR: New features, backward compatible
- PATCH: Bug fixes, backward compatible

### Release process
1. Create release branch from main: `release/v2.1.0`
2. Run full test suite including E2E
3. Update changelog and version numbers
4. Tag: `git tag -a v2.1.0 -m "Release v2.1.0"`
5. Deploy to staging, verify
6. Deploy to production
7. Merge release branch back to main

### Changelog
Keep a `CHANGELOG.md` updated with every release:
```markdown
## [2.1.0] - 2025-01-15
### Added
- User authentication via OAuth 2.0
### Fixed
- Payment timeout handling for slow networks
### Changed
- Upgrade to PostgreSQL 16
```

## Git Hooks (automate quality)

### Pre-commit
- Lint staged files (`lint-staged`)
- Format code (`prettier --write`)
- Run type check on changed files

### Commit-msg
- Validate conventional commit format (`commitlint`)

### Pre-push
- Run unit tests
- Check for secrets (`git-secrets`, `gitleaks`)