# Plan: Build an Agent-First Extendable Template Platform

The platform already has a solid two-tier foundation (`template-base` → `template-backend-java-spring-boot`) with reusable CI workflows, copilot instructions, and PR governance. The goal is to evolve this into a multi-tier, agent-first platform that can rapidly scaffold new projects, delegate work to AI agents with guardrails, and propagate updates downward across a template hierarchy.

---

## Progress Summary (updated 2026-02-22)

| Step | Status | Notes |
|------|--------|-------|
| 1. Clean up `.gitmodules` | ✅ Done | Stale root-level entry removed; five submodules under `templates/`. |
| 2. Multi-tier hierarchy | ✅ Done | `template-backend-java` (Tier 1) created; `docs/template-hierarchy.md` and `docs/overview.md` updated. |
| 3. Sync/propagation workflow | ✅ Done | `sync-templates.yml` fully implemented (241 lines); `docs/sync-workflow.md` complete. |
| 4. Agent delegation framework | ✅ Done | `agent-validation.yml` (283 lines), copilot instructions enhanced with Agent Delegation section, CODEOWNERS configured. |
| 5. Project scaffolding (`spin-up`) | ✅ Done | `create-project.yml` workflow created; `template-variables.yml` added to both archetype templates. |
| 6. Expand template catalog | ✅ Done | Python stack added: `template-backend-python` (Tier 1) + `template-backend-python-fastapi` (Tier 2). |

### PRs Pending Review

The following branches have been pushed and need PRs created/merged:

| Repo | Branch | Change |
|------|--------|--------|
| `template-backend-java-spring-boot` | `template-sync-gap-fix` | Adds `sync-config.yml` (Tier 2 sync chain) + `template-variables.yml` |
| `template-backend-java` | `template-sync-gap-fix` | Adds children list + `agent-validation.yml` to `sync_paths` |
| `template-base` | `add-python-stack` | Adds `template-backend-python` to children list |
| `template-platform` | `template-sync-gap-fix` | Adds `create-project.yml`, Python submodules, updated plan |

### Resolved Gaps

- ~~**`template-backend-java-spring-boot` missing `sync-config.yml`**~~ — Added via branch `template-sync-gap-fix`.
- ~~**`agent-validation.yml` not in Tier 1 sync_paths**~~ — Added to `template-backend-java/sync-config.yml` via branch `template-sync-gap-fix`.
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

**Status:** Complete. `create-project.yml` (290+ lines) in `template-platform` with 8 steps: clone archetype, resolve variables, substitute placeholders, rename Java/Python package directories, clean up template files, create GitHub repo via API, push code, and configure branch protection. Supports `template-backend-java-spring-boot` and `template-backend-python-fastapi` as template choices. `template-variables.yml` added to both archetype templates defining substitution variables, paths, and package rename rules. Uses clone + find-replace + push approach (not GitHub template API) to support variable substitution.

### 6. Expand the template catalog with at least one more stack

Add a second stack (e.g., `template-frontend-react` → `template-frontend-react-nextjs`, or `template-backend-python` → `template-backend-python-fastapi`) as submodules under `templates/` to validate that the hierarchy, sync, and agent-validation patterns work across different stacks.

**Status:** Complete. Python stack added as a second ecosystem:
- **`template-backend-python`** (Tier 1): `pyproject.toml` with ruff + mypy + pytest config, `.python-version`, CI calling base with `backend-tech-stack: python`, `dependabot.yml` for pip ecosystem, `sync-config.yml` declaring `template-base` as parent.
- **`template-backend-python-fastapi`** (Tier 2): FastAPI hello-world app with `/` and `/health` endpoints, Swagger/ReDoc auto-docs, async test suite using `httpx` + `pytest-asyncio`, `template-variables.yml` for scaffolding, `sync-config.yml` declaring `template-backend-python` as parent.
- Both repos created on GitHub, pushed, and added as submodules in `template-platform`.
- `template-base/sync-config.yml` updated (branch `add-python-stack`) to list `template-backend-python` as a child.

---

## Completed Actions

### A. ✅ Fixed `template-backend-java-spring-boot` sync gap
- Added `sync-config.yml` declaring Tier 2 parent and sync_paths.
- Added `template-variables.yml` for project scaffolding.
- Branch `template-sync-gap-fix` pushed, awaiting PR merge.

### B. ✅ Implemented project scaffolding workflow
- `create-project.yml` in `template-platform` with clone → substitute → push flow.
- Supports both Java Spring Boot and Python FastAPI archetypes.

### C. ✅ Added second stack (Python/FastAPI)
- `template-backend-python` (Tier 1) and `template-backend-python-fastapi` (Tier 2) created.
- Both repos pushed to GitHub and registered as submodules.

---

## Future Improvements (not in original plan)

1. **Frontend stack** — Add `template-frontend-react` (Tier 1) + `template-frontend-react-nextjs` (Tier 2) to cover frontend ecosystem.
2. **Automated CI validation on new projects** — Trigger CI on the newly created repo after scaffolding to verify green build.
3. **Template catalog discovery** — Add a `templates.json` manifest at the platform level listing all available archetypes with metadata (description, stack, variables).
4. **Copier/Cookiecutter integration** — If variable substitution needs grow beyond simple find-replace (conditional files, loops), evaluate Copier as a templating engine.

---

## Further Considerations

1. **Sync strategy** — Resolved: Actions + file-copy. Revisit Copier only if variable substitution (step 5) proves too complex with sed/envsubst.

2. **Agent identity** — Resolved: `github-actions[bot]` with `Co-authored-by` trailer for the delegating human.

3. **Hierarchy depth** — Resolved: Capped at three tiers. No evidence yet that a fourth tier is needed.

