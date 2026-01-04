# Pull Request Title Guidelines

## Overview

This repository enforces **Conventional Commits** format for pull request titles. When a PR is merged using squash or rebase merge, the PR title becomes the commit message in the main branch. Following this format ensures:

- **Automated changelog generation** - Tools can parse commit history to generate release notes
- **Semantic versioning** - Changes are categorized by type (feat, fix, etc.) to determine version bumps
- **Clear commit history** - Anyone reading the commit log can quickly understand what changed
- **Consistent conventions** - All contributors follow the same standard

## PR Title Format

Your PR title **must** match this pattern:

```
^(feat|fix|chore|docs|refactor|test|perf|ci|build)(\(.+\))?(!)?: .+
```

### Format Breakdown

```
<type>(<optional scope>)<optional !>: <description>
```

- **type** (required): The category of change
- **scope** (optional): The area of the codebase affected (e.g., `api`, `ui`, `auth`, `theme`)
- **!** (optional): Indicates a breaking change
- **description** (required): A clear, concise summary of the change

## Valid Commit Types

| Type | Description | Example Use Case |
|------|-------------|------------------|
| `feat` | A new feature | Adding a new API endpoint, new UI component |
| `fix` | A bug fix | Fixing a crash, correcting broken functionality |
| `chore` | Maintenance tasks | Updating dependencies, tooling changes |
| `docs` | Documentation changes | README updates, API documentation |
| `refactor` | Code restructuring without behavior change | Extracting functions, renaming variables |
| `test` | Adding or updating tests | New test cases, test framework updates |
| `perf` | Performance improvements | Optimization, caching implementation |
| `ci` | CI/CD pipeline changes | GitHub Actions workflows, build scripts |
| `build` | Build system or dependencies | Package.json, pom.xml, build configuration |

## Examples

### ✅ Valid PR Titles

```
feat: allow configuration of PR workflows
fix: correct typo in README
chore: update all dependencies to latest
docs: add PR title composition guidance
refactor: extract duplicate validation logic
test: add unit tests for auth service
perf: improve database query performance
ci: add PR title validation workflow
build: update Maven to version 3.9.5
```

### ✅ Valid PR Titles with Scope

```
feat(api): add user authentication endpoint
fix(ui): resolve button alignment issue
docs(theme): update homepage layout docs
refactor(auth): simplify token validation logic
test(api): add integration tests for user service
ci(workflows): enable parallel job execution
```

### ✅ Valid PR Titles with Breaking Change

```
feat!: migrate to new API v2
fix!: change response format to match spec
chore!: drop support for Node.js 14
refactor!: rename public API methods
```

### ❌ Invalid PR Titles

```
update README
my changes
add feature
Fixed a bug
WIP: new feature
[JIRA-123] Add authentication
Feature/add-login-page
Update dependencies and fix bug
```

**Why these are invalid:**
- Missing the type prefix (`feat`, `fix`, etc.)
- Capitalized description (should be lowercase after the colon)
- Multiple changes bundled together (should be separate PRs)
- Using project-specific prefixes instead of conventional commit types
- Using branch names as titles

## Breaking Changes

For changes that break backward compatibility, add an exclamation mark (`!`) after the type/scope:

```
feat!: remove deprecated API endpoints
fix(auth)!: change token format to JWT
chore!: upgrade to React 19 (requires migration)
```

This signals to automated tools that this is a **major version change**.

## Tips for Writing Good PR Titles

1. **Be specific** - "fix: correct null pointer in user service" is better than "fix: bug fix"
2. **Use imperative mood** - "add feature" not "added feature" or "adding feature"
3. **Keep it concise** - Aim for 50-72 characters
4. **No period at the end** - PR titles don't need punctuation
5. **One change per PR** - If you can't describe it with one type, split it into multiple PRs

## Common Mistakes

### Multiple Changes in One PR

❌ **Bad:**
```
feat: add login page and update dependencies
```

✅ **Good:** Split into two PRs:
```
feat: add login page
chore: update dependencies to latest
```

### Wrong Capitalization

❌ **Bad:**
```
Feat: Add new feature
feat: Add New Feature
```

✅ **Good:**
```
feat: add new feature
```

### Missing Colon or Space

❌ **Bad:**
```
feat add new feature
feat:add new feature
```

✅ **Good:**
```
feat: add new feature
```

## Checking Your PR Title

Before submitting your PR:

1. **Review the pattern** - Does your title match the format?
2. **Check the type** - Is the type appropriate for your change?
3. **Verify lowercase** - Is the description in lowercase?
4. **Ensure single focus** - Does the PR address one logical change?

If CI fails with a PR title validation error, refer back to this guide and update your PR title accordingly.

## Tools and Automation

This repository enforces PR title format through:

- **GitHub Actions** - Automated checks that validate PR titles using `.github/workflows/pr-title-lint.yml`
- **Branch protection rules** - Preventing merge until title is valid
- **Semantic release tools** - Automated versioning based on commit types

### Using the PR Title Lint Workflow

This repository provides a **reusable workflow** for PR title validation that can be used in derived repositories.

#### For Derived Repositories

Create `.github/workflows/pr-title-lint.yml` in your repository:

```yaml
name: PR Title Lint

on:
  pull_request:
    types:
      - opened
      - edited
      - synchronize
      - reopened

jobs:
  pr-title-lint:
    uses: EdwardRosenberg/template-base/.github/workflows/pr-title-lint.yml@main
```

This workflow will automatically validate PR titles against the Conventional Commits format and fail CI if the title doesn't conform.

#### Regex Pattern

The workflow validates PR titles using this regex pattern:

```
^(feat|fix|chore|docs|refactor|test|perf|ci|build)(\(.+\))?(!)?: .+
```

This ensures:
- Title starts with a valid type (feat, fix, chore, docs, refactor, test, perf, ci, build)
- Optional scope in parentheses
- Optional `!` for breaking changes
- Required colon and space
- Required description

## Additional Resources

- [Conventional Commits Specification](https://www.conventionalcommits.org/)
- [Semantic Versioning](https://semver.org/)
- [Example: Backstage PR Conventions](https://github.com/backstage/backstage/blob/master/CONTRIBUTING.md#creating-a-pull-request)
- [Example: Docker PR Conventions](https://github.com/docker/cli/blob/master/CONTRIBUTING.md#commit-messages)

## Questions?

If you're unsure about the correct type or format for your PR:

1. Review this guide and the examples above
2. Check recent merged PRs in the repository for reference
3. Ask in the PR comments for guidance from maintainers
