# Plan: Future Improvements for the Template Platform

The platform has completed its foundational 6-step plan: a three-tier hierarchy (base → stack → archetype), a sync workflow, an agent validation framework, a project scaffolding workflow, and two stacks (Java + Python). This plan captures the next wave of improvements to harden the platform, improve observability, and scale adoption.

## Current State (as of 2026-02-22)

```
template-base (Tier 0)
├── template-backend-java (Tier 1)
│   └── template-backend-java-spring-boot (Tier 2)
└── template-backend-python (Tier 1)
    └── template-backend-python-fastapi (Tier 2)
```

**What exists:**
- `sync-templates.yml` — staged parent→child file propagation via PRs
- `agent-validation.yml` — 4-gate reusable workflow (PR title, scope, build/test/lint, summary)
- `create-project.yml` — scaffold new repos from archetype templates with variable substitution
- `template-variables.yml` — per-archetype variable definitions for scaffolding
- `copilot-instructions.md` — agent delegation contract with task format and self-validation checklist
- `CODEOWNERS` — human review required on all PRs

**What's missing:** end-to-end validation, drift detection, security scanning, compliance reporting, catalog discoverability, and frontend/infra stack coverage.

## Steps

### 1. Add template integration tests

Create a workflow in `template-platform` that, on each push, clones every Tier 2 archetype template, runs its build + test suite, and reports the result. This catches situations where a `template-base` change (e.g., updated CI workflow inputs) silently breaks a downstream archetype.

- Add `.github/workflows/integration-tests.yml` in `template-platform`.
- For each Tier 2 submodule: checkout, install dependencies, run build command, run test command.
- Matrix strategy across all archetypes so failures are isolated per template.
- Trigger on pushes to `main` and on PRs that modify `templates/**` or `.github/workflows/`.
- Report results as a workflow summary and fail the run if any template is broken.

This is the foundation that makes everything else safe to iterate on. Without it, changes to `template-base` or the CI workflow are a leap of faith.

### 2. Add a template compliance report

Create a scheduled workflow that checks every child repo (via API) for expected files and configuration:

- Does the repo have `sync-config.yml`?
- Is `.editorconfig` identical to what the parent declares?
- Does `CODEOWNERS` exist?
- Is CI passing on `main`?
- Are there stale sync PRs older than N days?
- Does the repo have branch protection enabled?

Output a compliance matrix as a workflow artifact and optionally publish to GitHub Pages. This delivers on the "auditable governance" promise from the overview documentation.

### 3. Add post-scaffold CI verification to `create-project.yml`

After the scaffolding workflow pushes code to the new repo, add a step that:

- Triggers the CI workflow on the new repo via `workflow_dispatch` or waits for the automatic `push` trigger.
- Polls the workflow run status until completion (with a timeout).
- Reports the CI result in the workflow summary.
- If CI fails, prints the failure URL and marks the scaffold run as failed instead of silently creating a broken project.

### 4. Add sync drift detection

Add a scheduled workflow (e.g., weekly cron) in `template-platform` that:

- Runs `sync-templates.yml` in `dry-run` mode across all templates.
- Collects the list of templates with pending changes.
- Opens or updates a single tracking issue (e.g., `sync-drift-report`) with a summary table of which templates are out of sync and which files differ.
- Optionally posts to a Slack/Teams webhook.

This catches drift passively without requiring manual sync runs.

### 5. Add a security scanning gate to `agent-validation.yml`

Extend the agent validation workflow with an optional Gate 5:

- Run `gitleaks` on the PR diff to detect accidentally committed secrets.
- Run `osv-scanner` or `trivy` on dependency manifest files (`pom.xml`, `pyproject.toml`, `package.json`) to catch known vulnerabilities.
- Make both checks configurable via `require-secret-scan` and `require-dependency-scan` boolean inputs.
- Default `require-secret-scan` to `true` for agent PRs (agents are especially prone to introducing credentials from context).

### 6. Create a template catalog manifest

Create a `catalog.yml` at the platform root that lists all available archetype templates with metadata:

```yaml
templates:
  - name: template-backend-java-spring-boot
    description: "Spring Boot microservice with Swagger, health check, and Checkstyle"
    stack: java
    framework: spring-boot
    tier: 2
    repo: EdwardRosenberg/template-backend-java-spring-boot
    variables:
      - groupId
      - artifactId
      - basePackage
      - serviceName

  - name: template-backend-python-fastapi
    description: "FastAPI microservice with Swagger, health check, and ruff/mypy"
    stack: python
    framework: fastapi
    tier: 2
    repo: EdwardRosenberg/template-backend-python-fastapi
    variables:
      - projectName
      - serviceName
      - serviceDescription
```

Update `create-project.yml` to read the template list dynamically from `catalog.yml` instead of hardcoding the `choice` options. This also serves as a discoverable registry for humans and agents.

### 7. Add conflict-aware sync

Enhance `sync-templates.yml` to detect when a child has locally modified a synced file:

- Before copying from parent, compare the child's current version against the *previous* parent version (using the last sync commit or a stored hash).
- If the child's version differs from both the old and new parent versions, it means the child made local modifications that would be overwritten.
- In this case, label the PR as `sync/conflict` and add a comment explaining which files need manual merge.
- For non-conflicting files, proceed with the normal copy.

This prevents silent overwrites of legitimate local customizations.

### 8. Add a frontend stack (React/Next.js)

Add `template-frontend-react` (Tier 1) + `template-frontend-react-nextjs` (Tier 2) to validate cross-ecosystem support:

**Tier 1 — `template-frontend-react`:**
- `package.json` with scripts scaffold (no framework dependencies)
- `.nvmrc` pinning Node.js version
- ESLint + Prettier configuration
- `ci.yml` calling base with `frontend-enabled: true`, `frontend-tech-stack: node`
- `dependabot.yml` for npm ecosystem

**Tier 2 — `template-frontend-react-nextjs`:**
- Next.js App Router scaffold with TypeScript
- Tailwind CSS configuration
- Example page + layout
- Jest or Vitest test setup
- `template-variables.yml` for scaffolding

This exercises the CI workflow's `frontend-enabled` path which currently has no real consumer and proves the platform works across fundamentally different ecosystems (Node vs. JVM/Python).

### 9. Add an agent task queue via GitHub Issues

Define a formalized agent delegation flow:

- Create a standardized issue label: `agent-ready`.
- Define a required issue body format matching the copilot-instructions task description (Goal, Constraints, Acceptance Criteria, Files in Scope).
- Add an issue template (`.github/ISSUE_TEMPLATE/agent-task.yml`) to `template-base` that enforces the format.
- Create a workflow that watches for `agent-ready` issues, validates the task description format, and adds an `agent-validated` label when the format is correct.
- Future: integrate with Copilot Workspace or a custom orchestrator to auto-pick-up validated tasks.

This formalizes the "delegate to agent" flow beyond ad-hoc Copilot sessions.

### 10. Add changelog generation

Add a reusable workflow to `template-base` that auto-generates a `CHANGELOG.md` from Conventional Commit PR titles on release/tag:

- Trigger on `release` event or manual `workflow_dispatch`.
- Parse merged PR titles since the last release tag.
- Group by type (`feat`, `fix`, `chore`, etc.).
- Append to `CHANGELOG.md` with the release version and date.
- Commit and push (or open a PR with the changelog update).

Since PR title format is already enforced, this is low effort and high value.

### 11. Add scaffold smoke tests

Add a test job that validates the scaffolding pipeline without creating real repos:

- Run as part of PR CI for changes to `create-project.yml` or any `template-variables.yml`.
- Clone each archetype template.
- Substitute variables with test values (e.g., `groupId=com.test`, `artifactId=smoke-test`).
- Run the build + test suite on the substituted project.
- Verify all placeholder tokens were replaced (no remaining `com.example` or `hello-world-service`).
- Do NOT create a GitHub repo — purely local validation.

### 12. Add an infrastructure/IaC stack

Add `template-infra-terraform` (Tier 1) + `template-infra-terraform-aws` (Tier 2) to prove the platform can govern infrastructure-as-code with the same patterns:

**Tier 1 — `template-infra-terraform`:**
- Terraform version pinning (`.terraform-version`)
- `tflint` configuration
- `terraform fmt` and `terraform validate` in CI
- Backend configuration scaffold (S3 + DynamoDB for state)

**Tier 2 — `template-infra-terraform-aws`:**
- VPC + ECS/EKS module scaffold
- Environment-based workspace structure (`dev/`, `staging/`, `prod/`)
- Example `tfvars` files
- `template-variables.yml` for region, account ID, project name

## Priority Matrix

| Step | Feature | Effort | Impact | Priority |
|------|---------|--------|--------|----------|
| 1 | Template integration tests | Medium | High | **P0** |
| 2 | Compliance report | Low | High | **P0** |
| 3 | Post-scaffold CI verification | Low | Medium | **P1** |
| 4 | Sync drift detection | Low | Medium | **P1** |
| 5 | Security scanning gate | Low | High | **P1** |
| 6 | Template catalog manifest | Low | Medium | **P2** |
| 7 | Conflict-aware sync | Medium | Medium | **P2** |
| 8 | Frontend stack (React/Next.js) | Medium | Medium | **P2** |
| 9 | Agent task queue | High | High | **P3** |
| 10 | Changelog generation | Low | Low | **P3** |
| 11 | Scaffold smoke tests | Low | Low | **P3** |
| 12 | Infrastructure/IaC stack | Medium | Low | **P3** |

## Recommended Execution Order

**Phase 1 — Safety net (steps 1–2):**
Build the testing and compliance infrastructure so all subsequent changes can be validated automatically. Without integration tests, every change to `template-base` is a leap of faith.

**Phase 2 — Harden existing features (steps 3–5):**
Make the scaffolding workflow fail-safe, add passive drift detection, and add security scanning for agent PRs. These are low-effort improvements to features that already exist.

**Phase 3 — Scale the catalog (steps 6–8):**
Add discoverability, smarter sync, and a third ecosystem. This phase is about proving the platform scales beyond two stacks.

**Phase 4 — Advanced agent features (steps 9–12):**
Formalize agent task delegation, add changelog automation, scaffold testing, and infrastructure templates. These are higher-effort features that become valuable as adoption grows.

## Further Considerations

1. **GitHub App vs. PAT for sync and scaffolding** — The current `SYNC_TOKEN` PAT approach works but doesn't scale well (single credential, no granular permissions). Consider building a lightweight GitHub App that can be installed on all template repos with exactly the permissions needed (`contents: write`, `pull-requests: write`). This also provides better audit logging.

2. **Monorepo archetype** — Some teams may want a monorepo with both backend and frontend in one repo. Consider whether a Tier 2 archetype like `template-fullstack-java-react` makes sense, or whether the platform should recommend separate repos with a shared CI workflow.

3. **Template versioning** — Currently templates are always `@main`. As the catalog grows, consider tagging template releases (e.g., `v1.0.0`) so that the scaffolding workflow can pin to a specific version and downstream projects can opt in to updates at their own pace.

4. **Self-service template creation** — As the platform matures, provide a `create-template.yml` workflow that scaffolds a new Tier 1 or Tier 2 template with the correct `sync-config.yml`, governance files, and CI wiring already in place. This lowers the barrier for other teams to contribute new stacks.

