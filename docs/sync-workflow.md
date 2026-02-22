# Sync Workflow

The sync workflow (`sync-templates.yml`) automatically propagates changes from parent templates to their children across the three-tier hierarchy. It runs at the `template-platform` level and requires no changes to individual template repositories.

## How It Works

```
template-base (Tier 0)
    │
    │  sync-templates.yml detects changes here
    │  and opens a PR against ↓
    │
template-backend-java (Tier 1)
    │
    │  once merged, re-run sync to propagate ↓
    │
template-backend-java-spring-boot (Tier 2)
```

Sync is **staged** — it only propagates one tier at a time. Changes in `template-base` flow to Tier 1 children first. After those PRs are reviewed and merged, another sync run propagates to Tier 2.

## `sync-config.yml` Manifest

Every template (except `template-base`) declares:

```yaml
parent: EdwardRosenberg/template-backend-java   # GitHub repo slug of the parent
tier: 2                                          # 0=base, 1=stack, 2=archetype

sync_paths:                                      # Paths owned by the parent
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

The workflow reads `sync_paths` from each template's `sync-config.yml` and copies only those files from parent to child.

## Required Secret

The workflow needs a `SYNC_TOKEN` secret set in `template-platform` with `repo` scope (to push branches and open PRs on downstream template repos).

**Setting it up:**
1. Create a GitHub Personal Access Token (classic) with `repo` scope, or a fine-grained token with `contents: write` and `pull-requests: write` on each template repo.
2. Add it as a repository secret named `SYNC_TOKEN` in `template-platform`.

## Triggering Sync

### Automatic
The workflow triggers automatically on any push to `main` that touches `templates/**` or `.gitmodules` — i.e., when a template submodule pointer is updated.

### Manual (recommended for controlled rollouts)
Go to **Actions → Sync Templates → Run workflow** and provide:

| Input | Description |
|---|---|
| `source-repo` | Filter to a specific parent repo (e.g., `EdwardRosenberg/template-base`). Leave blank to sync all. |
| `dry-run` | Set to `true` to detect changes without opening PRs. Use this to preview what would change. |

**Recommended flow for rolling out a `template-base` change:**
1. Merge the change into `template-base`.
2. Update the submodule pointer in `template-platform`.
3. Run sync with `source-repo=EdwardRosenberg/template-base` and `dry-run=true` to preview.
4. Run sync with `dry-run=false` to open PRs against all Tier 1 children.
5. Review and merge each Tier 1 PR.
6. Run sync again with the updated Tier 1 repo as `source-repo` to propagate to Tier 2.

## What Gets Synced vs. What Doesn't

| Synced | Not synced |
|---|---|
| Files listed in `sync_paths` of the child's `sync-config.yml` | Any file not in `sync_paths` |
| Org-wide governance files (CI, PR template, copilot instructions) | Stack-specific config (pom.xml, checkstyle.xml for Tier 1→2 unless declared) |
| `.editorconfig`, `.gitignore` | Application source code |
| Issue templates | README.md |

## Adding a New Template to Sync

1. Create the new template repo with a `sync-config.yml` declaring its parent and `sync_paths`.
2. Add it as a submodule under `templates/` in `template-platform`.
3. Update the parent template's `sync-config.yml` `children` list (used for documentation; the workflow discovers children via `sync-config.yml` scanning).
4. Trigger a manual sync with `dry-run=true` to validate the configuration.

