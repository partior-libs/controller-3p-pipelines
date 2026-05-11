# controller-3p-pipelines

> Reusable GitHub Actions workflows for third-party (3P) component CI/CD pipelines — GoQuorum, Tessera, Quorum plugins, and related blockchain tooling.

## Overview

`controller-3p-pipelines` is the centralized pipeline controller for building, versioning, and publishing third-party open-source components that Partior forks or maintains. It manages the CI for Go-based (go-ethereum/GoQuorum), Gradle-based (Tessera), and plugin-based (Quorum Account Plugin, Quorum Signer Plugin) projects, along with Docker image builds and artifact promotion to JFrog Artifactory.

## Architecture

All workflows use a **config-driven** pattern:
1. `read-repo` — extracts branch/PR metadata
2. `read-config` — pulls per-project YAML config from the `projects/` directory via `gcs-yaml-importer`
3. `ci-build` — runs versioning (`gcs-versioning-bot`), builds, tests, Docker publish, artifact signing
4. `package-helmfile` — packages helmfile artifacts and uploads to Artifactory

Project-specific configuration lives in `projects/<repo-name>.yaml` with fallback to `projects/<type>-default.yaml`.

## Prerequisites

| Requirement | Details |
|-------------|---------|
| GitHub App | `BOT_APP_ID` variable + `BOT_APP_KEY` secret (for cross-repo checkout) |
| JFrog Artifactory | `JFROG_ARTIFACTORY_CONFIG` secret |
| JIRA | `JIRA_USERNAME` + `JIRA_PASSWORD` secrets (for versioning bot) |
| GPG signing | `GPG_SECRET_KEYS` + `GPG_OWNERTRUST` secrets (for Go/Gradle builds) |
| Partior-stable approval | GitHub environment `PARTIOR-STABLE` configured with required reviewers |

## Usage

Reference a workflow from your repository:

```yaml
# In your-repo/.github/workflows/ci.yaml
jobs:
  ci:
    uses: partior-libs/controller-3p-pipelines/.github/workflows/pipeline-template-go-ci-quorum.yaml@partior-stable
    with:
      CONTROLLER-REPO: partior-libs/controller-3p-pipelines
      PROJECT-REPO: partior-quorum/quorum   # your 3P project repo
    secrets: inherit
```

```yaml
# For Gradle/Tessera builds
jobs:
  ci:
    uses: partior-libs/controller-3p-pipelines/.github/workflows/pipeline-template-gradle-ci-tessera.yaml@partior-stable
    with:
      CONTROLLER-REPO: partior-libs/controller-3p-pipelines
      PROJECT-REPO: partior-quorum/tessera
    secrets: inherit
```

```yaml
# For Quorum plugin CI
jobs:
  ci:
    uses: partior-libs/controller-3p-pipelines/.github/workflows/pipeline-template-quorum-plugin-ci.yaml@partior-stable
    with:
      CONTROLLER-REPO: partior-libs/controller-3p-pipelines
      PROJECT-REPO: partior-quorum/quorum-account-plugin-hashicorp-vault
    secrets: inherit
```

## Workflows

| Workflow File | Trigger | Description |
|--------------|---------|-------------|
| `controller-ci.yaml` | push/PR to `main` | Self-CI for this controller repo — runs `pipeline-template-controller-ci.yaml` |
| `pipeline-template-controller-ci.yaml` | `workflow_call` | Versions, packages, and tags this controller as a helmfile artifact |
| `pipeline-template-docker-ci.yml` | `workflow_call` | Docker image CI — build, tag, push to Artifactory |
| `pipeline-template-go-ci-quorum.yaml` | `workflow_call` | Full CI for Go-based 3P components (GoQuorum/go-ethereum) |
| `pipeline-template-gradle-ci.yaml` | `workflow_call` | Full CI for Gradle-based 3P components |
| `pipeline-template-gradle-ci-tessera.yaml` | `workflow_call` | Full CI for Tessera (Gradle + Docker) |
| `pipeline-template-quorum-plugin-ci.yaml` | `workflow_call` | CI for Quorum Account Plugins |
| `pipeline-template-quorum-signer-plugin-ci.yaml` | `workflow_call` | CI for Quorum Signer Plugin (Hashicorp Vault) |
| `pipeline-template-update-partior-stable.yaml` | `workflow_call` | Moves `partior-stable` tag with approval gate |
| `update-partior-stable.yaml` | `workflow_dispatch` | Manual trigger to promote a ref to `partior-stable` |

## Configuration

### Inputs for `pipeline-template-go-ci-quorum.yaml` / Gradle templates

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `CONTROLLER-REPO` | This controller repository | Yes | — |
| `CONTROLLER-REF` | Branch/tag of the controller | No | `partior-stable` |
| `PROJECT-REPO` | The 3P project repository to build | Yes | — |

### Inputs for `pipeline-template-update-partior-stable.yaml`

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `TARGET-REFERENCE` | Tag or ref to promote to `partior-stable` | Yes | — |
| `SKIP-APPROVAL` | Bypass the approval gate | No | `false` |

### Secrets (inherited via `secrets: inherit`)

| Secret | Description |
|--------|-------------|
| `BOT_APP_KEY` | GitHub App private key |
| `JFROG_ARTIFACTORY_CONFIG` | JFrog auth config |
| `JIRA_USERNAME` / `JIRA_PASSWORD` | Jira integration for versioning |
| `GPG_SECRET_KEYS` / `GPG_OWNERTRUST` | GPG artifact signing |

## Project Structure

```
controller-3p-pipelines/
├── .github/
│   └── workflows/
│       ├── controller-ci.yaml                         # Self-CI
│       ├── pipeline-template-controller-ci.yaml       # Controller versioning/packaging
│       ├── pipeline-template-docker-ci.yml            # Docker CI template
│       ├── pipeline-template-go-ci-quorum.yaml        # Go/GoQuorum CI template
│       ├── pipeline-template-gradle-ci.yaml           # Gradle CI template
│       ├── pipeline-template-gradle-ci-tessera.yaml   # Tessera CI template
│       ├── pipeline-template-quorum-plugin-ci.yaml    # Quorum plugin CI
│       ├── pipeline-template-quorum-signer-plugin-ci.yaml
│       ├── pipeline-template-update-partior-stable.yaml
│       └── update-partior-stable.yaml                 # Manual partior-stable promotion
└── projects/
    ├── controller-default.yaml                        # Default controller CI config
    ├── docker-default.yaml                            # Default Docker CI config
    ├── 3p-golang-quorum-default.yaml                  # Default Go CI config
    ├── 3p-gradle-default.yaml                         # Default Gradle CI config
    ├── quorum.yaml                                    # GoQuorum project config
    ├── tessera.yaml                                   # Tessera project config
    └── *.yaml                                         # Per-project pipeline configs
```

## Contributing

1. Add or update a project config in `projects/<repo-name>.yaml`
2. Test changes on a feature branch before merging to `main`
3. Use `update-partior-stable.yaml` (workflow_dispatch) to promote `main` → `partior-stable` after approval
4. Commit format: `<TICKET_NUMBER> <message>` (e.g., `DSO-123 Add quorum signer support`)

## License

See [LICENSE.md](./LICENSE.md).
