# Examples

Complete, copy-ready pipeline examples for deploying .NET applications to Azure using [NuvTools Pipelines](../README.md).

Each example is a self-contained consumer reference — copy the `.github/workflows/pipeline.yml` into your repository, adjust the parameters, and you're ready to go.

## Overview

| Example | Target | Mode | Description |
|---|---|---|---|
| [aks-full](aks-full/) | AKS | Helm + Docker (in-repo values) | Full pipeline with per-environment Helm values |
| [aks-gitops](aks-gitops/) | AKS | Helm + Docker (external config repo) | AKS pipeline with GitOps config repository |
| [appservice-basic](appservice-basic/) | App Service | Zip Deploy | Artifact-based deploy with slot swap |
| [appservice-docker](appservice-docker/) | App Service | Docker | Container image deploy with slot swap |
| [pipeline-repo](pipeline-repo/) | — | — | GitOps configuration repository structure |

## How to Choose

### Where are you deploying?

- **AKS (Kubernetes)** — Use `aks-full` or `aks-gitops`
- **App Service** — Use `appservice-basic` or `appservice-docker`

### Do you need Docker?

- **No** — Use `appservice-basic` (simplest setup, no ACR needed)
- **Yes, deploying to AKS** — Use `aks-full` or `aks-gitops`
- **Yes, deploying to App Service** — Use `appservice-docker`

### Where should configuration live?

- **In the app repo** (simpler) — Use `aks-full` or `appservice-basic`/`appservice-docker`
- **In a separate config repo** (GitOps) — Use `aks-gitops` + `pipeline-repo`

### Decision flowchart

```
Start
  |
  +-- Deploying to AKS?
  |     +-- Yes --> Config in app repo?
  |     |             +-- Yes --> aks-full
  |     |             +-- No  --> aks-gitops + pipeline-repo
  |     +-- No  --> Deploying to App Service?
  |           +-- Yes --> Need Docker?
  |                         +-- Yes --> appservice-docker
  |                         +-- No  --> appservice-basic
```

## Common Setup Steps

All examples share the same prerequisites:

1. **Azure Authentication** — Set up OIDC federated credentials ([authentication guide](../docs/authentication-setup.md))
2. **Repository Secrets** — Add `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`
3. **GitHub Environments** — Create `dev`, `staging`, `production` environments with appropriate approval rules

## Orchestration Flow

All examples follow the same version resolution strategy:

| Git Event | Version | Environment | Action |
|---|---|---|---|
| Push to `main` | `dev{run_number}` | `dev` | Build + deploy |
| Tag `v*-alpha*`/`v*-beta*`/`v*-rc*` | SemVer | `staging` | Build + deploy |
| Tag `v*` (stable) | SemVer | `production` | Build + deploy |
| Pull request | `pr{number}-{sha7}` | — | Build + test only |
