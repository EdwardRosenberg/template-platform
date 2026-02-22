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

## Template Hierarchy

The Template Platform uses a **three-tier architecture** that separates governance, stack tooling, and application scaffolding into distinct layers. For a full reference, see [docs/template-hierarchy.md](template-hierarchy.md).

### Tier 0 — Base Template (`template-base`)
The **base template** is the single source of truth for organization-wide standards. It contains:
- `.github/` - Shared GitHub Actions workflows and configurations
- `.editorconfig` - Consistent code formatting across all projects
- `.gitignore` - Common ignore patterns
- `CODEOWNERS` - Code review requirements
- `SECURITY.md` - Security policies and reporting procedures
- `CONTRIBUTING.md` - Contribution guidelines

`template-base` is **never used directly** to create projects. It is the foundation that all Tier 1 stack templates inherit from.

### Tier 1 — Stack Templates
**Stack templates** are language- or ecosystem-specific templates that extend the base template. They add build tooling and quality gates for a technology stack but remain app-agnostic. Examples:
- `template-backend-java` - Maven wrapper, Java conventions, CI wired for Java
- `template-frontend-react` - Node.js config, ESLint/Prettier, CI wired for Node
- `template-data-python` - pyproject.toml, ruff/mypy, CI wired for Python

Stack templates are **not used directly** to create application projects. They serve as parents for Tier 2 archetype templates.

### Tier 2 — Archetype Templates
**Archetype templates** extend a stack template with an opinionated, runnable application scaffold. Examples:
- `template-backend-java-spring-boot` - Spring Boot hello-world app, Swagger setup
- `template-frontend-react-nextjs` - Next.js scaffold with Tailwind
- `template-data-python-fastapi` - FastAPI scaffold with example router

Each archetype template:
1. Inherits governance from `template-base` (via its stack template parent)
2. Adds framework-specific boilerplate and a working application
3. Can be used directly via GitHub's "Use this template" to create new projects with CI green from day one

## Why Does Sync Exist?

When organizational standards evolve (new security policies, updated workflows, improved tooling), **every existing project needs those updates**.

Without sync, you face two bad options:
1. **Manual updates**: Copy changes to dozens of repositories by hand (error-prone, time-consuming)
2. **Stale repos**: Accept that old projects diverge from current standards (security risk, technical debt)

### How Sync Works

Sync is implemented as a GitHub Actions workflow at `.github/workflows/sync-templates.yml` in this repository. The process:

1. Reads `sync-config.yml` from each template submodule to build a parent→child propagation map
2. Detects which `sync_paths` differ between parent and child
3. Opens a reviewable PR against each child template with only the changed files
4. Ensures changes are never destructive — only explicitly declared paths are touched

Each template declares its parent and the paths it accepts in its own `sync-config.yml`. The workflow can be triggered automatically on push or manually via `workflow_dispatch` with an optional dry-run mode.

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

1. **Tier 0 (`template-base`)** = The style guide and shared standards
2. **Tier 1 (Stack Templates)** = Technology section editors (Java, React, Python)
3. **Tier 2 (Archetype Templates)** = Ready-to-use article templates for a specific framework
4. **Sync** = The distribution system that propagates updates down the hierarchy
5. **Your Projects** = Individual articles created from archetype templates

When the style guide changes, all stack templates get updated automatically, and all archetype templates and projects inherit the new standards through staged propagation.

## Quick Reference

- **Creating a new project**: Use an archetype template (Tier 2), e.g., `template-backend-java-spring-boot`
- **Updating shared standards**: Modify `template-base`, then run sync
- **Adding a new tech stack**: Create a Tier 1 stack template inheriting from `template-base`
- **Adding a new framework**: Create a Tier 2 archetype template inheriting from a stack template
- **Understanding the hierarchy**: See [docs/template-hierarchy.md](template-hierarchy.md)
- **Understanding what's shared**: Check the `shared_paths` in sync configuration

## Next Steps

For more detailed information, refer to these guides:

- **[Template Hierarchy](template-hierarchy.md)** - Detailed three-tier model, decision guide, and instructions for adding new stacks
- **[Sync Workflow](sync-workflow.md)** - How sync works, required secrets, and how to trigger it
- **Using Templates** - How to create new projects from archetype templates (coming soon)
- **Running Sync** - Step-by-step guide for propagating updates (coming soon)

---

**Questions?** This platform is designed to be simple and predictable. If something is unclear after reading this document, that's a bug in the documentation—please file an issue.
