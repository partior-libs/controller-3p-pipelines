# controller-3p-pipelines
> CI/CD pipeline templates for 3rd-party upstream components used by Partior's platform.

## Overview

This controller repository hosts reusable GitHub Actions workflow templates for building, testing, and releasing **3rd-party (upstream) open-source components** that Partior depends on — including GoQuorum, Tessera, Quorum plugins, and Docker images. Other repositories call these templates via the `uses:` directive, pointing to the `partior-stable` ref.

## Usage

Reference any workflow template from a calling repository:

```yaml
jobs:
  ci:
    uses: partior-libs/controller-3p-pipelines/.github/workflows/pipeline-template-gradle-ci.yaml@partior-stable
    with:
      CONTROLLER-REPO: partior-libs/controller-3p-pipelines
      PROJECT-REPO: partior-quorum/tessera
    secrets: inherit
```

## Workflows

| Workflow File | Purpose |
|---|---|
| `pipeline-template-controller-ci.yaml` | CI for the controller repo itself |
| `pipeline-template-docker-ci.yml` | Docker image build & push for 3p components |
| `pipeline-template-go-ci-quorum.yaml` | Go CI pipeline for GoQuorum fork |
| `pipeline-template-gradle-ci-tessera.yaml` | Gradle CI for Tessera (privacy manager) |
| `pipeline-template-gradle-ci.yaml` | Generic Gradle CI for 3p Java components |
| `pipeline-template-quorum-plugin-ci.yaml` | CI for Quorum plugin builds |
| `pipeline-template-quorum-signer-plugin-ci.yaml` | CI for Quorum signer plugin |
| `pipeline-template-update-partior-stable.yaml` | Promotes the `partior-stable` tag to latest commit |

## Inputs / Parameters

### `pipeline-template-gradle-ci.yaml` (common inputs)

| Input | Required | Default | Description |
|---|---|---|---|
| `CONTROLLER-REPO` | ✅ | — | This controller repo (`partior-libs/controller-3p-pipelines`) |
| `CONTROLLER-REF` | ❌ | `main` | Branch/tag of the controller to use |
| `PROJECT-REPO` | ✅ | — | The 3p source repo to build |
| `RUNNER` | ❌ | `self-hosted-dev` | Runner label pool |

### `pipeline-template-go-ci-quorum.yaml` (common inputs)

| Input | Required | Default | Description |
|---|---|---|---|
| `CONTROLLER-REPO` | ✅ | — | This controller repo |
| `PROJECT-REPO` | ✅ | — | GoQuorum fork repository |
| `RUNNER` | ❌ | `self-hosted-dev` | Runner label pool |

## Outputs

| Output | Description |
|---|---|
| `artifact-version` | Built artifact version string |
| `source-branch` | Branch that triggered the pipeline |
| `commit-sha` | Full commit SHA of the build |

## Prerequisites

**Required secrets** (inherited via `secrets: inherit`):

| Secret | Purpose |
|---|---|
| `JFROG_ARTIFACTORY_TOKEN` | Publish artifacts to JFrog |
| `GPG_SECRET_KEYS` | GPG signing key for release artifacts |
| `GPG_OWNERTRUST` | GPG owner-trust data |
| `GH_PAT_TOKEN` | GitHub PAT for cross-repo operations |
| `PARTIOR_GITHUB_APP_KEY` | GitHub App private key |

**Infrastructure:**
- Self-hosted GitHub Actions runners (`self-hosted-dev`, `self-hosted-prod`)
- JFrog Artifactory at `https://partior.jfrog.io/artifactory`

## Contributing

```bash
git commit -m "<TICKET_NUMBER> <COMMIT_MESSAGE>"
# Example:
git commit -m "DSO-123 Add Go module cache to quorum CI"
```

Changes to workflow templates are validated by `pipeline-template-controller-ci.yaml` on every pull request.

## License

Proprietary — Copyright © Partior Pte. Ltd. All rights reserved.
