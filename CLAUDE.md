# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

NuvTools Pipelines is a reusable GitHub Actions workflow and composite action library for deploying .NET applications to Azure (AKS and App Service). It is not a typical application — there is no build step, no package.json, and no application code. The deliverables are YAML workflows, composite actions, Helm charts, and Dockerfiles consumed by other repositories via `uses:` references.

## Validation Commands

```bash
yamllint .github/            # Validate YAML syntax
actionlint                   # Validate GitHub Actions workflows
helm lint charts/default     # Validate the default Helm chart
```

There are no unit tests. Integration testing is done by referencing a fork branch from a consumer repository.

## Architecture

### Reusable Workflows (`.github/workflows/`)

- **ci.yml** — CI pipeline (artifact): .NET build/test/coverage, publishes build artifacts; outputs `version`
- **ci-docker.yml** — CI pipeline (container): .NET build/test/coverage, Docker build & push to ACR; outputs `version` and `image-uri`
- **cd-aks.yml** — CD to AKS: Helm-based deployment with placeholder substitution, optional GitOps config repo
- **cd-appservice.yml** — CD to App Service: zip deploy or Docker container deploy, optional slot swap

### Composite Actions (`.github/actions/`)

Four building blocks that workflows compose together:

| Action | Role |
|---|---|
| `azure-login` | OIDC auth (preferred) with Service Principal fallback |
| `dotnet-build-test` | Restore, build, test, coverage report, publish, artifact upload |
| `docker-build-push` | Build Docker image and push to ACR |
| `helm-deploy` | Chart prep, placeholder substitution, Helm upgrade, rollout verify |

### Environment Selection Strategy

Pipelines run on manual `workflow_dispatch`. The operator picks the target environment from a `choice` dropdown input, and a `runDeploy` boolean gates the deploy job. There is no automatic environment inference from git refs (the old `resolve-version` action and its tag/branch/PR mapping were removed).

- The `environment` input maps to a GitHub Environment, so approval gates and environment secrets apply to the deploy job.
- The CI workflows derive the version/image tag from an optional input (`app-version` / `image-tag`), defaulting to `${{ github.run_number }}`.

### Helm Chart (`charts/default/`)

Generic chart with Deployment, Service, ConfigMap (env vars), HPA, Ingress, ServiceAccount. Placeholder tokens `<APP_NAME>`, `<APP_VERSION>`, and `<IMAGE_PATH>` are substituted at deploy time by the `helm-deploy` action.

### Default Dockerfile (`dockerfiles/dotnet.Dockerfile`)

Multi-stage ASP.NET Core image exposing port 8080 (non-root). Configurable .NET version and entry point DLL.

### GitOps Config Repo Pattern

Optionally, Helm values and App Service settings live in a separate config repository (per-environment files like `values-dev.yaml`, `values-staging.yaml`, `values-production.yaml`). The `config-repo` input and `CONFIG_REPO_TOKEN` secret enable this.

## Code Conventions

- **YAML**: 2-space indentation, no trailing whitespace
- **Actions**: every input/output must have a `description`
- **Workflows**: include comments for non-obvious logic
- **Helm charts**: follow [Helm best practices](https://helm.sh/docs/chart_best_practices/)
- **Commits**: [Conventional Commits](https://www.conventionalcommits.org/) — `feat:`, `fix:`, `docs:`, `chore:`, `refactor:`, `test:`
- **Changelog**: [Keep a Changelog](https://keepachangelog.com/) format in `CHANGELOG.md`
