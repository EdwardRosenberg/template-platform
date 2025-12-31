# Template Platform Overview

## What is the Template Platform?

The Template Platform is a system for managing shared code and configuration across multiple repositories. It ensures consistency, accelerates development velocity, and enforces governance across all projects in your organization.

## Why Do Templates Exist?

Templates solve several critical challenges in modern software development:

### 1. **Consistency**
Without templates, each project reinvents the wheel for common concerns like:
- CI/CD pipelines and workflows
- Security policies and code ownership
- Code quality standards (linting, formatting)
- Documentation structure
- Contribution guidelines

Templates ensure every project starts with the same battle-tested foundations.

### 2. **Velocity**
Starting a new project should take minutes, not days. Templates provide:
- Pre-configured build pipelines
- Security scanning and compliance checks
- Standard project structure
- Ready-to-use workflows

Engineers can focus on business logic instead of infrastructure setup.

### 3. **Governance**
Organizations need control over:
- Security policies (SECURITY.md, CODEOWNERS)
- Compliance requirements
- Quality standards
- Operational best practices

Templates make governance enforceable and auditable across all repositories.

## Base vs Leaf Templates

The Template Platform uses a two-tier architecture:

### Base Template (`template-base`)
The **base template** is the single source of truth for shared organizational standards. It contains:
- `.github/` - Shared GitHub Actions workflows and configurations
- `.editorconfig` - Consistent code formatting across all projects
- `.gitignore` - Standard ignore patterns
- `CODEOWNERS` - Code review requirements
- `SECURITY.md` - Security policies and reporting procedures
- `CONTRIBUTING.md` - Contribution guidelines

The base template is **never used directly** to create projects. Instead, it serves as the foundation that all leaf templates inherit from.

### Leaf Templates
**Leaf templates** are technology-specific templates that extend the base template. Examples include:
- `template-backend-spring` - For Spring Boot microservices
- `template-frontend-react` - For React applications
- `template-data-pipeline` - For data engineering projects

Each leaf template:
1. Inherits shared files from the base template
2. Adds technology-specific boilerplate and configuration
3. Can be used directly via GitHub's "Use this template" feature to create new projects

## Why Does Sync Exist?

When organizational standards evolve (new security policies, updated workflows, improved tooling), **every existing project needs those updates**.

Without sync, you face two bad options:
1. **Manual updates**: Copy changes to dozens of repositories by hand (error-prone, time-consuming)
2. **Stale repos**: Accept that old projects diverge from current standards (security risk, technical debt)

### How Sync Works

The sync process:
1. Detects changes in the base template's shared paths
2. Automatically propagates those changes to all leaf templates
3. Creates reviewable pull requests for each update
4. Ensures changes are never destructive (only specified paths are touched)

This keeps the entire ecosystem up-to-date with minimal manual effort.

## What Problems Does This Solve?

### Problem 1: "Every repo is a snowflake"
**Without templates**: Each project has unique CI, different security policies, inconsistent tooling.  
**With templates**: Standard, predictable structure across all repositories.

### Problem 2: "Setting up a new project takes days"
**Without templates**: Engineers spend time researching and configuring infrastructure.  
**With templates**: Click "Use this template" and start coding in minutes.

### Problem 3: "Security updates require touching 50+ repositories"
**Without sync**: Manual updates to every repo (often skipped due to effort).  
**With sync**: Automated propagation with reviewable PRs.

### Problem 4: "No visibility into compliance across repos"
**Without templates**: Policies exist in documents, not code.  
**With templates**: Governance is encoded in version-controlled files.

## Mental Model Summary

Think of the Template Platform as a **publishing system**:

1. **Base Template** = The style guide and shared content
2. **Leaf Templates** = Section templates (Sports, Politics, Business)
3. **Sync** = The distribution system that pushes updates to all sections
4. **Your Projects** = Individual articles created from the section templates

When the style guide changes, all sections get updated automatically, and all future articles inherit the new standards.

## Quick Reference

- **Creating a new project**: Use a leaf template (e.g., `template-backend-spring`)
- **Updating shared standards**: Modify the base template, then run sync
- **Adding a new tech stack**: Create a new leaf template that inherits from base
- **Understanding what's shared**: Check the `shared_paths` in sync configuration

## Next Steps

For more detailed information, refer to these guides (coming soon):

- **Using Templates** - How to create new projects from leaf templates
- **Sync Rules** - What files are synchronized and what's excluded
- **Running Sync** - How to propagate updates from base to leaf templates

---

**Questions?** This platform is designed to be simple and predictable. If something is unclear after reading this document, that's a bug in the documentation—please file an issue.
