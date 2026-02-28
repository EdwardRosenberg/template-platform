# Template Hierarchy

The Template Platform uses a **three-tier hierarchy** that separates organizational governance, stack-specific tooling, and opinionated application boilerplate into distinct layers. Each tier inherits from the one above it and adds only what is relevant to its level of abstraction.

## Tier Overview

```
Tier 0 — template-base
    │
    ├── Tier 1 — template-backend-java          (stack template)
    │       └── Tier 2 — template-backend-java-spring-boot   (archetype template)
    │
    ├── Tier 1 — template-backend-python        (stack template)
    │       └── Tier 2 — template-backend-python-fastapi     (archetype template)
    │
    └── Tier 1 — template-frontend-react        (stack template, planned)
            └── Tier 2 — template-frontend-react-nextjs      (archetype template, planned)
```

---

## Tier 0 — Base Template (`template-base`)

**Purpose:** Organization-wide governance and standards.

This tier contains everything that **every project in the organization** must have, regardless of language or framework.

### What belongs here
- `.github/workflows/` — Reusable CI/CD workflows (`ci.yml`, `pr-title-lint.yml`, `agent-validation.yml`)
- `.github/copilot-instructions.md` — AI agent guardrails and coding conventions
- `.github/CODEOWNERS` — Default code review requirements
- `.github/PULL_REQUEST_TEMPLATE.md` — Standard PR description format
- `.github/ISSUE_TEMPLATE/` — Standard issue templates
- `.editorconfig` — Universal formatting rules
- `.gitignore` — Common ignore patterns
- `SECURITY.md` — Security policy and vulnerability reporting
- `CONTRIBUTING.md` — Contribution guidelines

### What does NOT belong here
- Language-specific build tooling (Maven, Gradle, npm)
- Framework configuration
- Application source code or boilerplate
- Stack-specific linters or test runners

### Usage
`template-base` is **never used directly** to create application projects. It is used as the foundation for Tier 1 stack templates.

---

## Tier 1 — Stack Templates

**Purpose:** Stack-specific tooling and conventions, without application boilerplate.

Stack templates inherit everything from `template-base` and layer on the tooling required for a particular technology stack. They remain **app-agnostic** — they configure the build system and quality gates but do not include a runnable application.

### Examples

| Template | Stack | Adds |
|---|---|---|
| `template-backend-java` | Java (any framework) | Maven wrapper, Java `.gitignore` additions, `ci.yml` calling base with `backend-tech-stack: java`, Checkstyle config |
| `template-backend-python` | Python (any framework) | `pyproject.toml`, `ruff` / `mypy` config, pytest setup, `ci.yml` calling base with `backend-tech-stack: python` |
| `template-frontend-react` | React / TypeScript (planned) | Node.js `.nvmrc`, `package.json` scaffold, ESLint + Prettier configs, `ci.yml` calling base with `frontend-tech-stack: node` |

### What belongs here
- Build system configuration (Maven `pom.xml` parent, `package.json` scripts, `pyproject.toml`)
- Language-specific `.gitignore` additions
- Linter and formatter configuration for the stack
- `ci.yml` that calls the base reusable workflow with the correct stack inputs
- `dependabot.yml` for the relevant package ecosystem
- `template-variables.yml` — Declares the variables a derived archetype must supply (e.g., `groupId`, `artifactId`)

### What does NOT belong here
- A runnable "hello world" application
- Framework-specific annotations, controllers, or handlers
- Application configuration files (e.g., `application.properties`)

### Usage
Stack templates are **not used directly** for new application projects. They are the parent for Tier 2 archetype templates.

---

## Tier 2 — Archetype Templates

**Purpose:** Opinionated, runnable application scaffolding for a specific framework within a stack.

Archetype templates inherit from a Tier 1 stack template and add the framework-specific boilerplate needed to have a **working, deployable application** from day one.

### Examples

| Template | Parent Stack Template | Adds |
|---|---|---|
| `template-backend-java-spring-boot` | `template-backend-java` | Spring Boot parent POM, `HelloWorldController`, `application.properties`, Springdoc/Swagger setup |
| `template-backend-python-fastapi` | `template-backend-python` | FastAPI app scaffold, example router, `uvicorn` configuration |
| `template-frontend-react-nextjs` | `template-frontend-react` (planned) | Next.js app scaffold, Tailwind config, example page and layout |

### What belongs here
- A minimal but runnable application (`HelloWorld`-style)
- Framework parent POM or dependency baseline
- Framework-specific configuration (`application.properties`, `next.config.ts`)
- Integration test scaffolding for the framework
- `README.md` with stack-specific "getting started" instructions

### Usage
Archetype templates are **used directly** (via GitHub's "Use this template") to create new application projects. They provide a working build with CI green on day one.

---

## Inheritance and Sync

Each tier propagates changes **downward** to its direct children only. The sync mechanism (see `docs/overview.md`) ensures that changes made in a parent template are automatically proposed to child templates via pull requests.

### Propagation example

1. A new reusable workflow is added to `template-base`.
2. The sync workflow opens a PR against `template-backend-java` and `template-backend-python` (Tier 1 children).
3. Once merged, the sync workflow opens a PR against `template-backend-java-spring-boot` and `template-backend-python-fastapi` (Tier 2 children).

This staged propagation keeps each tier responsible for reviewing only what changed at its level.

### `sync-config.yml` manifest

Each template declares its parent and the paths it accepts from sync in a `sync-config.yml` at its root. See `docs/overview.md` for the sync configuration reference.

---

## Decision Guide: Where Does a File Belong?

Use this checklist when deciding which tier a new file should live in:

| Question | If yes → belongs in |
|---|---|
| Does every project need this, regardless of stack? | Tier 0 (`template-base`) |
| Is this specific to a language/ecosystem but not a framework? | Tier 1 (stack template) |
| Is this specific to a single framework or requires a running app? | Tier 2 (archetype template) |
| Is this project-specific (team name, service URL, DB schema)? | The generated project itself |

---

## Adding a New Stack

To add a new technology stack to the platform:

1. **Create a Tier 1 stack template** repository (e.g., `template-frontend-react`).
   - Add it as a submodule under `templates/` in this repository.
   - Inherit `.github/` and shared config files from `template-base`.
   - Add stack-specific build tooling and a `ci.yml` calling the base reusable workflow.
   - Add a `sync-config.yml` pointing to `template-base` as the parent.

2. **Create one or more Tier 2 archetype templates** (e.g., `template-frontend-react-nextjs`).
   - Add it as a submodule under `templates/`.
   - Add a `sync-config.yml` pointing to the Tier 1 stack template as the parent.
   - Include a minimal runnable application scaffold.

3. **Register both in `sync-config.yml`** at the platform root so the sync workflow knows the parent→child relationship.

4. **Validate** by running the platform's sync workflow in dry-run mode and confirming CI is green on both new templates.

