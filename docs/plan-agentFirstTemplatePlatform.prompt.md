# Plan: Build an Agent-First Extendable Template Platform

The platform already has a solid two-tier foundation (`template-base` → `template-backend-java-spring-boot`) with reusable CI workflows, copilot instructions, and PR governance. The goal is to evolve this into a multi-tier, agent-first platform that can rapidly scaffold new projects, delegate work to AI agents with guardrails, and propagate updates downward across a template hierarchy.

---

## Progress Summary (updated 2025-02-22)

| Step | Status | Notes |
|------|--------|-------|
| 1. Clean up `.gitmodules` | ✅ Done | Stale root-level entry removed; three submodules under `templates/`. |
| 2. Multi-tier hierarchy | ✅ Done | `template-backend-java` (Tier 1) created; `docs/template-hierarchy.md` and `docs/overview.md` updated. |
| 3. Sync/propagation workflow | ✅ Done | `sync-templates.yml` fully implemented (241 lines); `docs/sync-workflow.md` complete. |
| 4. Agent delegation framework | ✅ Done | `agent-validation.yml` (283 lines), copilot instructions enhanced with Agent Delegation section, CODEOWNERS configured. |
| 5. Project scaffolding (`spin-up`) | ❌ Not started | No `create-project.yml` or `template-variables.yml` exists. |
| 6. Expand template catalog | ❌ Not started | Only the Java stack exists. No second stack template. |

### Open Gaps in Completed Steps

- **`template-backend-java-spring-boot` is missing `sync-config.yml`** — The Tier 2 archetype has no sync manifest, so the sync workflow cannot propagate changes from `template-backend-java` down to it. A `sync-config.yml` declaring `parent: EdwardRosenberg/template-backend-java` and appropriate `sync_paths` must be added (via PR to the Spring Boot template repo).
- **`agent-validation.yml` not in Tier 1 sync_paths** — `template-base/sync-config.yml` lists `agent-validation.yml` but `template-backend-java/sync-config.yml` does not include it. This means the agent-validation workflow will not auto-propagate from base → Tier 1 children unless the Tier 1 sync_paths are updated, or the approach is intentional (Tier 1 templates call the reusable workflow from `template-base@main` directly instead of copying the file).
- **Decisions resolved** — Agent identity: settled on `github-actions[bot]` + `Co-authored-by` trailers. Sync strategy: Actions + file-copy (no Copier). Hierarchy depth: capped at three tiers.

---

## Steps

### 1. Clean up `.gitmodules` and establish `templates/` as the canonical submodule root ✅

Remove the stale top-level `template-base` entry from `.gitmodules` (lines 1–3 reference `path = template-base` outside `templates/`). Ensure only `templates/template-base` and `templates/template-backend-java-spring-boot` remain.

**Status:** Complete. `.gitmodules` now lists three submodules: `templates/template-base`, `templates/template-backend-java`, and `templates/template-backend-java-spring-boot`. Commit `f7083ed`.

### 2. Introduce a multi-tier template hierarchy with an intermediate "stack" layer ✅

Currently the docs describe a flat base→leaf model. Add a middle tier between `template-base` and final leaf templates:

- **Tier 0 — `template-base`**: Org-wide governance (CI, PR lint, copilot instructions, CODEOWNERS, editorconfig).
- **Tier 1 — Stack templates** (new): e.g., `template-backend-java`, `template-frontend-react`, `template-data-python`. These inherit from base and add stack-specific tooling (Maven wrapper, Spring parent POM, Node/TS config, pytest setup) but remain app-agnostic.
- **Tier 2 — Archetype templates** (current leaf, renamed): e.g., `template-backend-java-spring-boot`, `template-frontend-react-nextjs`. These inherit from the stack template and add opinionated boilerplate (hello-world app, API scaffolding).

Document this in a new `docs/template-hierarchy.md` and update `docs/overview.md` to replace the "Base vs Leaf" section.

**Status:** Complete. `template-backend-java` repo created as Tier 1 stack template with Maven wrapper, `checkstyle.xml`, `sync-config.yml`, and CI wiring. `docs/template-hierarchy.md` (158 lines) documents tiers, decision guide, and how to add new stacks. `docs/overview.md` (145 lines) updated with three-tier model.

### 3. Build a sync/propagation workflow in `template-platform` ✅

Create `.github/workflows/sync-templates.yml` at the platform level that:

- Detects changes pushed to `template-base` (or any tier-N template).
- Opens PRs against all downstream tier-(N+1) templates with the changed shared-paths.
- Uses a `sync-config.yml` manifest in each template repo (read via GitHub API) that declares parent→child relationships and which paths propagate.
- This replaces the conceptual "sync" described in the overview with a concrete, runnable mechanism.

**Status:** Complete. `sync-templates.yml` (241 lines) reads `sync-config.yml` from each template via the GitHub Contents API, builds a matrix of parent→child pairs, copies changed `sync_paths`, and opens timestamped PRs. Supports `dry-run` and `source-repo` filter inputs. `docs/sync-workflow.md` (100 lines) documents usage, secrets, and the staged propagation model.

### 4. Create an agent delegation and validation framework ✅

Add to `template-base` (so every derived repo inherits it):

- **`.github/workflows/agent-validation.yml`**: A reusable workflow that runs on PRs authored by bots/agents. It enforces: build passes, tests pass, no new lint warnings, PR title lint, and a configurable set of custom checks.
- **`.github/copilot-instructions.md` enhancements**: Add an "Agent Delegation" section with structured task-description format (goal, constraints, acceptance criteria, files-in-scope) that agents must follow. Add instructions for agents to self-validate before submitting (run build, run tests, check for scope creep).
- **`CODEOWNERS` pattern**: Require a human reviewer on agent-authored PRs by adding a `* @org/human-reviewers` rule as a safety net.

**Status:** Complete. `agent-validation.yml` (283 lines) in `template-base` with four gates: PR title lint, scope check (allowed-paths), build/test/lint, and summary gate. Supports Java, Python, Node stacks. Copilot instructions enhanced (251 lines) with Agent Delegation section (task description format, self-validation checklist, identity guidance, prohibited actions). `CODEOWNERS` configured with `* @EdwardRosenberg/platform-team`.

### 5. Add a project scaffolding CLI or workflow (`spin-up`)

Create a `.github/workflows/create-project.yml` (workflow_dispatch) at the platform level that:

- Accepts inputs: project name, template to use (from the hierarchy), target GitHub org/owner.
- Uses GitHub API to create a new repo from the selected template.
- Injects project-specific values (group ID, artifact ID, service name) via a `template-variables.yml` manifest in each template.
- Configures branch protection, required status checks, and CODEOWNERS automatically.
- Outputs a ready-to-clone repo URL with CI already green.

**Status:** Not started.

**Pre-requisites before implementation:**
1. Add `template-variables.yml` to each archetype template (Tier 2) defining the variables that must be substituted (e.g., `groupId`, `artifactId`, `serviceName`, `packageName`).
2. Add `sync-config.yml` to `template-backend-java-spring-boot` first (gap from step 2/3).
3. Decide whether to use GitHub's "Use this template" API or clone + re-init approach. GitHub's template API is simpler but does not support variable substitution — recommend clone + find-replace + push.

### 6. Expand the template catalog with at least one more stack

Add a second stack (e.g., `template-frontend-react` → `template-frontend-react-nextjs`, or `template-backend-python` → `template-backend-python-fastapi`) as submodules under `templates/` to validate that the hierarchy, sync, and agent-validation patterns work across different stacks.

**Status:** Not started.

**Recommended next stack:** `template-backend-python` (Tier 1) + `template-backend-python-fastapi` (Tier 2). Python is the second most common backend stack and validates the multi-language support in `agent-validation.yml` (which already has `tech-stack: python` support). Alternatively, a frontend stack (`template-frontend-react`) would validate cross-ecosystem support more broadly.

---

## Recommended Next Actions (in priority order)

### A. Fix the `template-backend-java-spring-boot` sync gap (small, high-impact)
Add a `sync-config.yml` to the `template-backend-java-spring-boot` repo declaring:
```yaml
parent: EdwardRosenberg/template-backend-java
tier: 2
sync_paths:
  - .github/workflows/ci.yml
  - .github/workflows/pr-title-lint.yml
  - .github/copilot-instructions.md
  - .github/PULL_REQUEST_TEMPLATE.md
  - .github/ISSUE_TEMPLATE/bug.yml
  - .github/ISSUE_TEMPLATE/feature.yml
  - .github/ISSUE_TEMPLATE/chore.yml
  - .editorconfig
  - .gitignore
  - checkstyle.xml
```
This completes the end-to-end sync chain (base → java → spring-boot) and is required before step 5 can work correctly.

### B. Implement step 5 — project scaffolding workflow
This is the highest-value remaining step. It turns the platform from a "documentation + CI" system into an active project creation tool.

### C. Implement step 6 — second stack template
Validates extensibility. Can be done in parallel with step 5 if desired.

---

## Further Considerations

1. **Sync strategy** — Resolved: Actions + file-copy. Revisit Copier only if variable substitution (step 5) proves too complex with sed/envsubst.

2. **Agent identity** — Resolved: `github-actions[bot]` with `Co-authored-by` trailer for the delegating human.

3. **Hierarchy depth** — Resolved: Capped at three tiers. No evidence yet that a fourth tier is needed.

