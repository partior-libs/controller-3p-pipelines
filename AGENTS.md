# AGENTS.md — controller-3p-pipelines

## Purpose

`controller-3p-pipelines` is a GitHub Actions reusable-workflow controller that provides standardised CI/CD pipeline templates for 3rd-party upstream components integrated into Partior's DevSecOps platform. It targets GoQuorum (Go), Tessera (Gradle/Java), Quorum plugins, and Docker image builds, ensuring consistent build, test, sign, and publish steps across all upstream forks and dependencies.

## Repository Map

| Path | Role |
|---|---|
| `.github/workflows/pipeline-template-controller-ci.yaml` | Self-CI for this controller repo |
| `.github/workflows/pipeline-template-docker-ci.yml` | Docker build/push template |
| `.github/workflows/pipeline-template-go-ci-quorum.yaml` | Go build/test for GoQuorum fork |
| `.github/workflows/pipeline-template-gradle-ci-tessera.yaml` | Gradle build for Tessera |
| `.github/workflows/pipeline-template-gradle-ci.yaml` | Generic Gradle CI |
| `.github/workflows/pipeline-template-quorum-plugin-ci.yaml` | Quorum plugin CI |
| `.github/workflows/pipeline-template-quorum-signer-plugin-ci.yaml` | Quorum signer plugin CI |
| `.github/workflows/pipeline-template-update-partior-stable.yaml` | Promote `partior-stable` tag |
| `.github/workflows/controller-ci.yaml` | Internal controller validation |
| `.github/workflows/update-partior-stable.yaml` | Alternate stable-tag updater |

## Tech Stack

| Tool | Version / Notes |
|---|---|
| GitHub Actions | `workflow_call` reusable workflows |
| Go | As required by GoQuorum upstream |
| Gradle | As required by Tessera upstream |
| Docker | Buildx multi-platform |
| JFrog Artifactory | `https://partior.jfrog.io/artifactory` |
| `partior-libs/gcs-pipe-utils` | Shared pipeline utility actions (`@partior-stable`) |
| `partior-libs/gcs-versioning-bot` | Semantic versioning action |

## Architecture Patterns

- All workflows are declared with `on: workflow_call` — they are **never triggered directly**.
- Callers pin to `@partior-stable`, a floating tag that this controller promotes via `pipeline-template-update-partior-stable.yaml`.
- Runner selection is delegated to `gcs-pipe-utils/actions/generic-runner-selection` before any build job.
- Secrets are propagated via `secrets: inherit` from the calling workflow.
- The controller fetches its own config from `CONTROLLER-REPO` at `CONTROLLER-REF` to drive per-project build settings (YAML-driven configuration).

## Development Commands

```bash
# Validate YAML syntax of all workflow files
find .github/workflows -name "*.yaml" -o -name "*.yml" | xargs -I{} sh -c 'echo "Checking {}..." && python3 -c "import yaml,sys; yaml.safe_load(open(sys.argv[1]))" {}'

# List all workflow_call inputs across templates
grep -A3 'required:' .github/workflows/pipeline-template-*.yaml

# Check which actions are pinned to partior-stable
grep -r 'partior-stable' .github/workflows/
```

## Environment Setup

| Variable / Secret | Required | Purpose |
|---|---|---|
| `JFROG_ARTIFACTORY_TOKEN` | ✅ | Artifact publishing |
| `GPG_SECRET_KEYS` | ✅ | GPG artifact signing |
| `GPG_OWNERTRUST` | ✅ | GPG trust chain |
| `GH_PAT_TOKEN` | ✅ | Cross-repo GitHub API calls |
| `PARTIOR_GITHUB_APP_KEY` | ✅ | GitHub App authentication |
| `DOCKER_REGISTRY_BASE_URL` | Set in workflow env | `https://partior.jfrog.io/artifactory` |

## Coding Conventions

- Workflow filenames: `pipeline-template-<stack>-<purpose>.yaml` (or `.yml` for legacy Docker/runner files)
- Input names: `UPPER-KEBAB-CASE` for `workflow_call` inputs
- Environment variables: `UPPER_SNAKE_CASE`
- All reusable workflows must declare `on: workflow_call` as the sole trigger
- Version pins for third-party actions must use commit SHAs in production; `partior-stable` for internal actions

## Testing

1. **Syntax validation:** Parse all YAML files with `python3 -c "import yaml; yaml.safe_load(open('file.yaml'))"`
2. **Controller CI:** `pipeline-template-controller-ci.yaml` runs on every PR against this repo
3. **Dry-run:** Call the template from a test repository with `RUNNER: ubuntu-latest` to validate the workflow graph without self-hosted runners
4. **Diff review:** Use `gh workflow view` to inspect workflow structure before merging

## Key Abstractions

| Concept | Description |
|---|---|
| `partior-stable` | Floating Git tag; callers pin to this for stable pipeline versions |
| Controller Repo | Holds the workflow templates; consumed via `uses:` |
| Project Repo | The 3p source code repository being built |
| Runner Selection | Dynamic runner pool selection via `gcs-pipe-utils` |
| YAML Config Importer | Pattern that loads per-repo build config from the controller at runtime |

## Agentic Task Guidance

✅ **Safe to do:**
- Add new `pipeline-template-*.yaml` workflow files
- Update action version pins in existing workflows
- Modify runner selection logic
- Update documentation

⚠️ **Use caution:**
- Changing shared `env:` blocks — affects all jobs in the workflow
- Modifying `pipeline-template-update-partior-stable.yaml` — directly controls the stable tag promotion
- Altering input/output names — breaking change for all callers

❌ **Do not do:**
- Remove or rename existing `workflow_call` inputs without a migration plan
- Hardcode secrets or tokens in workflow files
- Add `on: push` or `on: pull_request` triggers to template files
- Commit directly to `main` without a PR

## External Dependencies

| Dependency | Used By |
|---|---|
| `partior-libs/gcs-pipe-utils@partior-stable` | Runner selection, utilities |
| `partior-libs/gcs-versioning-bot@partior-stable` | Semantic version calculation |
| `partior-libs/gcs-generic-packager@partior-stable` | Artifact packaging |
| `actions/checkout@v4` | Source checkout |
| `actions/upload-artifact` / `download-artifact` | Artifact passing between jobs |
| JFrog Artifactory | Artifact storage |

## Common Pitfalls

- **`partior-stable` tag mismatch:** If `update-partior-stable.yaml` fails, callers will use a stale version. Monitor the tag update job.
- **Runner availability:** `self-hosted-dev` runners may be busy during peak hours; builds will queue silently.
- **YAML config not found:** If `CONTROLLER-REPO` or `CONTROLLER-REF` is wrong, the config importer step fails with a cryptic error.
- **Go module cache:** GoQuorum CI caches are keyed on `go.sum`; changing dependencies without updating the cache key causes cache misses.
- **Gradle daemon conflicts:** Parallel Gradle builds on the same runner can cause port conflicts; use `--no-daemon` in CI.
