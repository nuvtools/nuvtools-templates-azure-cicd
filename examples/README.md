# Examples

Complete, copy-ready pipeline examples for deploying .NET applications to Azure using [NuvTools Pipelines](../README.md).

Each example is a self-contained consumer reference — copy the `.github/workflows/pipeline.yml` into your repository, adjust the parameters, and you're ready to go.

## Overview

| Example | Target | Mode | Description |
|---|---|---|---|
| [aks-full](aks-full/) | AKS | Helm + Docker | Full pipeline with per-environment Helm values |
| [appservice-basic](appservice-basic/) | App Service | Zip Deploy | Artifact-based deploy with slot swap |
| [appservice-docker](appservice-docker/) | App Service | Docker | Container image deploy with slot swap |

## How to Choose

### Where are you deploying?

- **AKS (Kubernetes)** — Use `aks-full`
- **App Service** — Use `appservice-basic` or `appservice-docker`

### Do you need Docker?

- **No** — Use `appservice-basic` (simplest setup, no ACR needed)
- **Yes, deploying to AKS** — Use `aks-full`
- **Yes, deploying to App Service** — Use `appservice-docker`

### Decision flowchart

```
Start
  |
  +-- Deploying to AKS?
  |     +-- Yes --> aks-full
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

All examples run on manual `workflow_dispatch`. Run the workflow from the **Actions** tab and pick the target environment from the dropdown:

| Dispatch input | Version | Environment | Action |
|---|---|---|---|
| `environment: dev`, `runDeploy: true` | run number | `dev` | Build + deploy |
| `environment: staging`, `runDeploy: true` | run number | `staging` | Build + deploy |
| `environment: production`, `runDeploy: true` | run number | `production` | Build + deploy |
| `runDeploy: false` | run number | — | Build + test only |
