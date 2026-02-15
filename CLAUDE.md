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

- **ci.yml** — CI pipeline: version resolution, .NET build/test/coverage, Docker build & push to ACR
- **cd-aks.yml** — CD to AKS: Helm-based deployment with placeholder substitution, optional GitOps config repo
- **cd-appservice.yml** — CD to App Service: zip deploy or Docker container deploy, optional slot swap

### Composite Actions (`.github/actions/`)

Five building blocks that workflows compose together:

| Action | Role |
|---|---|
| `resolve-version` | Maps git refs to version strings and environment names |
| `azure-login` | OIDC auth (preferred) with Service Principal fallback |
| `dotnet-build-test` | Restore, build, test, coverage report, publish, artifact upload |
| `docker-build-push` | Build Docker image and push to ACR |
| `helm-deploy` | Chart prep, placeholder substitution, Helm upgrade, rollout verify |

### Version Resolution Strategy

Central to the pipeline — `resolve-version` maps git events to versions and environments:

- `refs/heads/main|master` → `dev{run_number}` → environment `dev`
- `refs/tags/v*-alpha|beta|rc` → semver (no `v` prefix) → environment `staging`
- `refs/tags/v*` (stable) → semver (no `v` prefix) → environment `production`
- `refs/pull/*` → `pr{number}-{sha7}` → CI only (no deploy)
- Other branches → `{branch-slug}-{sha7}` → CI only (no deploy)

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
