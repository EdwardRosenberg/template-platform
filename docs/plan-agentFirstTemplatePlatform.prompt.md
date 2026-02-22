# Plan: Build an Agent-First Extendable Template Platform

The platform already has a solid two-tier foundation (`template-base` → `template-backend-java-spring-boot`) with reusable CI workflows, copilot instructions, and PR governance. The goal is to evolve this into a multi-tier, agent-first platform that can rapidly scaffold new projects, delegate work to AI agents with guardrails, and propagate updates downward across a template hierarchy.

## Steps

### 1. Clean up `.gitmodules` and establish `templates/` as the canonical submodule root

Remove the stale top-level `template-base` entry from `.gitmodules` (lines 1–3 reference `path = template-base` outside `templates/`). Ensure only `templates/template-base` and `templates/template-backend-java-spring-boot` remain.

### 2. Introduce a multi-tier template hierarchy with an intermediate "stack" layer

Currently the docs describe a flat base→leaf model. Add a middle tier between `template-base` and final leaf templates:

- **Tier 0 — `template-base`**: Org-wide governance (CI, PR lint, copilot instructions, CODEOWNERS, editorconfig).
- **Tier 1 — Stack templates** (new): e.g., `template-backend-java`, `template-frontend-react`, `template-data-python`. These inherit from base and add stack-specific tooling (Maven wrapper, Spring parent POM, Node/TS config, pytest setup) but remain app-agnostic.
- **Tier 2 — Archetype templates** (current leaf, renamed): e.g., `template-backend-java-spring-boot`, `template-frontend-react-nextjs`. These inherit from the stack template and add opinionated boilerplate (hello-world app, API scaffolding).

Document this in a new `docs/template-hierarchy.md` and update `docs/overview.md` to replace the "Base vs Leaf" section.

### 3. Build a sync/propagation workflow in `template-platform`

Create `.github/workflows/sync-templates.yml` at the platform level that:

- Detects changes pushed to `template-base` (or any tier-N template).
- Opens PRs against all downstream tier-(N+1) templates with the changed shared-paths.
- Uses a `sync-config.yml` manifest in the platform root that maps parent→children relationships and declares which paths propagate (e.g., `.github/workflows/ci.yml`, `.editorconfig`, `copilot-instructions.md`).
- This replaces the conceptual "sync" described in the overview with a concrete, runnable mechanism.

### 4. Create an agent delegation and validation framework

Add to `template-base` (so every derived repo inherits it):

- **`.github/workflows/agent-validation.yml`**: A reusable workflow that runs on PRs authored by bots/agents. It enforces: build passes, tests pass, no new lint warnings, PR title lint, and a configurable set of custom checks.
- **`.github/copilot-instructions.md` enhancements**: Add an "Agent Delegation" section with structured task-description format (goal, constraints, acceptance criteria, files-in-scope) that agents must follow. Add instructions for agents to self-validate before submitting (run build, run tests, check for scope creep).
- **`CODEOWNERS` pattern**: Require a human reviewer on agent-authored PRs by adding a `* @org/human-reviewers` rule as a safety net.

### 5. Add a project scaffolding CLI or workflow (`spin-up`)

Create a `.github/workflows/create-project.yml` (workflow_dispatch) at the platform level that:

- Accepts inputs: project name, template to use (from the hierarchy), target GitHub org/owner.
- Uses GitHub API to create a new repo from the selected template.
- Injects project-specific values (group ID, artifact ID, service name) via a `template-variables.yml` manifest in each template.
- Configures branch protection, required status checks, and CODEOWNERS automatically.
- Outputs a ready-to-clone repo URL with CI already green.

### 6. Expand the template catalog with at least one more stack

Add a second leaf template (e.g., `template-frontend-react-nextjs` or `template-backend-python-fastapi`) as a submodule under `templates/` to validate that the hierarchy, sync, and agent-validation patterns work across different stacks. This proves the platform is truly extensible.

## Further Considerations

1. **Sync strategy: GitHub Actions + template repos as remotes vs. a dedicated sync tool (e.g., Copier, Cookiecutter)?** Actions-based approach keeps everything in-platform; Copier adds templating power (variable substitution, conditional files) but introduces a dependency. Recommend starting with Actions + simple file-copy, then evaluate Copier if variable substitution becomes a bottleneck.

2. **Agent identity: Should agents commit as a bot account (e.g., `github-actions[bot]`) or as the delegating user?** Bot account makes it easy to filter agent PRs for the validation workflow; user account preserves attribution. Recommend a dedicated bot account with `Co-authored-by` trailer for the human who delegated.

3. **How deep should the hierarchy go?** Three tiers (base → stack → archetype) covers most cases. Going deeper (e.g., base → stack → archetype → project-type) adds sync complexity. Recommend capping at three tiers initially and revisiting only if real use cases demand it.

