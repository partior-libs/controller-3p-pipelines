# AGENTS.md — controller-3p-pipelines

## Purpose

Reusable GitHub Actions workflow controller for building, versioning, and publishing third-party (3P) blockchain components (GoQuorum/go-ethereum, Tessera, Quorum plugins). Consumed by `partior-quorum/*` 3P component repositories to trigger CI pipelines. Config-driven via YAML files in `projects/`.

## Repository Map

| Path | Role |
|------|------|
| `.github/workflows/controller-ci.yaml` | Self-CI: versions and packages this controller |
| `.github/workflows/pipeline-template-controller-ci.yaml` | Core controller versioning/helmfile packaging template |
| `.github/workflows/pipeline-template-docker-ci.yml` | Docker image build/push template |
| `.github/workflows/pipeline-template-go-ci-quorum.yaml` | Go/GoQuorum component CI template |
| `.github/workflows/pipeline-template-gradle-ci.yaml` | Gradle component CI template |
| `.github/workflows/pipeline-template-gradle-ci-tessera.yaml` | Tessera-specific Gradle+Docker CI |
| `.github/workflows/pipeline-template-quorum-plugin-ci.yaml` | Quorum Account Plugin CI |
| `.github/workflows/pipeline-template-quorum-signer-plugin-ci.yaml` | Quorum Signer Plugin CI |
| `.github/workflows/pipeline-template-update-partior-stable.yaml` | `partior-stable` tag promotion with approval gate |
| `.github/workflows/update-partior-stable.yaml` | Manual `partior-stable` promotion trigger |
| `projects/controller-default.yaml` | Default controller CI configuration |
| `projects/docker-default.yaml` | Default Docker CI configuration |
| `projects/3p-golang-quorum-default.yaml` | Default Go CI config |
| `projects/3p-gradle-default.yaml` | Default Gradle CI config |
| `projects/<repo>.yaml` | Per-project pipeline overrides |

## Tech Stack

| Component | Version/Details |
|-----------|-----------------|
| GitHub Actions | Reusable workflows (`workflow_call`) |
| Go | 1.22 (GoQuorum builds) |
| Gradle | Project-defined (Tessera) |
| Docker | `partior.jfrog.io` registry |
| JFrog Artifactory | `3p-docker-dev` repo, `partior.jfrog.io` |
| `gcs-yaml-importer` | `partior-libs/gcs-yaml-importer@main` (config importer) |
| `gcs-versioning-bot` | `partior-libs/gcs-versioning-bot@main` (version management) |
| `gcs-pipe-utils` | `partior-libs/gcs-pipe-utils@partior-stable` |
| GPG | Artifact signing for Go/Gradle releases |

## Architecture Patterns

- **GitHub Reusable Workflow Library** — all `pipeline-template-*.yaml` files use `on: workflow_call`
- **Config-driven pipelines** — behaviour controlled entirely by `projects/<repo>.yaml` YAML files
- **Staged job DAG**: `read-repo` → `read-config` → `ci-build` → `package-helmfile`
- **`partior-stable` tag pattern** — stable production reference, separate from `main`
- **Approval gate** — moving `partior-stable` requires environment approval in `pipeline-template-update-partior-stable.yaml`

## Development Commands

| Command | Description |
|---------|-------------|
| Edit `projects/<repo>.yaml` | Configure pipeline behavior for a specific 3P project |
| Trigger `update-partior-stable.yaml` | Promote `main` to `partior-stable` (requires approval) |
| Push to `main` | Triggers `controller-ci.yaml` self-CI |

## Environment Setup

| Secret/Variable | Description | Example |
|----------------|-------------|---------|
| `BOT_APP_ID` (var) | GitHub App ID for cross-repo auth | `12345` |
| `BOT_APP_KEY` (secret) | GitHub App private key | PEM key |
| `JFROG_ARTIFACTORY_CONFIG` | JFrog auth config JSON | `{"username":"...","password":"..."}` |
| `JIRA_USERNAME` | Jira username for versioning bot | `svc-account@partior.com` |
| `JIRA_PASSWORD` | Jira password for versioning bot | — |
| `GPG_SECRET_KEYS` | Base64 GPG private keys | — |
| `GPG_OWNERTRUST` | GPG owner trust file | — |

## Coding Conventions

- All workflow template files start with `pipeline-template-`
- Dispatch entry points (not templates) are named without the `pipeline-template-` prefix
- Inputs use `UPPER-KEBAB-CASE` for GitHub Actions inputs
- Environment variables use `SNAKE_CASE` in `env:` blocks
- Job names use `kebab-case`
- `YAML_CONFIG_IMPORTER` and `YAML_BRANCH_CONFIG_IMPORTER` are run-scoped artifact names
- All workflows use `secrets: inherit` — secrets are not explicitly listed in templates
- `CONTROLLER-REPO` + `CONTROLLER-REF` inputs follow the universal convention across all Partior controllers

## Testing

- No dedicated test framework for workflow YAML
- `controller-ci.yaml` acts as the integration test: it validates the controller packages successfully
- Test pipeline changes on a feature branch before merging to `main`
- Use `workflow_dispatch` on feature branches to test templates manually

## Key Abstractions

| File | Purpose |
|------|---------|
| `pipeline-template-controller-ci.yaml` | The universal "package this controller as a versioned artifact" template |
| `pipeline-template-go-ci-quorum.yaml` | Full CI for Go/go-ethereum forks: checkout, build, Docker, sign, push |
| `pipeline-template-gradle-ci-tessera.yaml` | Full CI for Tessera: Gradle build + Quorum Docker image |
| `projects/*.yaml` | Per-project overrides — controls version bump rules, branch behaviors, signing, deployment flags |

## Agentic Task Guidance

- ✅ Safe: Reading `projects/*.yaml` files, reading workflow YAML, checking workflow inputs/outputs
- ✅ Safe: Adding a new `projects/<new-repo>.yaml` config following existing structure
- ✅ Safe: Updating default behavior in `*-default.yaml` project files
- ⚠️ Review: Modifying `pipeline-template-controller-ci.yaml` — affects all other controllers that call it
- ⚠️ Review: Changing `pipeline-template-update-partior-stable.yaml` — affects production tag promotion across the platform
- ⚠️ Review: Changing Go version or Gradle versions — verify against upstream 3P project requirements
- ❌ Avoid: Committing secrets to `projects/*.yaml` files
- ❌ Avoid: Removing the `CONTROLLER-REF` default `partior-stable` without updating all callers
- ❌ Avoid: Triggering `update-partior-stable.yaml` without verifying `controller-ci.yaml` passed on the target commit

## External Dependencies & Integrations

| Dependency | Usage |
|-----------|-------|
| `partior-libs/gcs-yaml-importer` | Config parsing — reads `projects/` YAML files |
| `partior-libs/gcs-versioning-bot` | Semantic versioning, Jira ticket integration, Artifactory version tracking |
| `partior-libs/gcs-pipe-utils` | Runner selection, generic utility actions |
| `actions/create-github-app-token@v1` | GitHub App token generation for cross-repo access |
| JFrog Artifactory | `partior.jfrog.io` — artifact storage, Docker registry |
| Jira | Version tracking via ticket numbers in commit messages |

## Common Pitfalls

- **`CONTROLLER-REPO` must match the repo containing `projects/<calling-repo>.yaml`** — if a downstream repo calls a template from here, it must pass `CONTROLLER-REPO: partior-libs/controller-3p-pipelines`
- **`partior-stable` vs `main`**: Templates at `@partior-stable` are production-safe; use `@main` only for testing changes
- **GPG secrets must be base64-encoded** in the `GPG_SECRET_KEYS` secret
- **`YAML_CONFIG_IMPORTER` artifacts** are scoped per run (`${{ github.run_id }}_${{ github.run_number }}`); they must be uploaded before being downloaded in dependent jobs
- **`projects/<repo>.yaml` must exist** for the calling project repo name — missing config causes pipeline failure at the `read-config` job
