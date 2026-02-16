# AKS GitOps (External Config Repo)

Complete pipeline for deploying a .NET application to AKS using an external GitOps repository for Helm values.

## What It Demonstrates

- CI with build, test, Docker build and push to ACR
- CD to AKS via Helm with values stored in a separate repository (config repo)
- Automatic deployment to dev, staging and production based on git refs
- Usage of the `CONFIG_REPO_TOKEN` secret for config repo access

## Two-Repository Pattern

```
my-app/                          (application repository)
  src/
  tests/
  .github/workflows/pipeline.yml  <-- this example

my-pipeline-repo/                (configuration repository)
  my-app/
    values-dev.yaml
    values-staging.yaml
    values-production.yaml
    env-values-dev.yaml
    env-values-staging.yaml
    env-values-production.yaml
```

See [examples/pipeline-repo/](../pipeline-repo/) for the config repo structure.

## Prerequisites

- AKS cluster per environment (dev, staging, production)
- Azure Container Registry (ACR)
- App Registration with OIDC federated credentials ([authentication guide](../../docs/authentication-setup.md))
- GitOps configuration repository with Helm values
- Personal Access Token (PAT) with read access to the config repo

## Required Secrets

| Secret | Description |
|---|---|
| `AZURE_CLIENT_ID` | App Registration Client ID |
| `AZURE_TENANT_ID` | Azure AD Tenant ID |
| `AZURE_SUBSCRIPTION_ID` | Azure Subscription ID |
| `CONFIG_REPO_TOKEN` | PAT with read permission on the config repo |

## Orchestration Flow

| Action | Result |
|---|---|
| Push to `main` | Build + deploy to **dev** |
| Tag `v1.0.0-rc.1` | Build + deploy to **staging** |
| Tag `v1.0.0` | Build + deploy to **production** |
| Pull request | Build + test only (no deploy) |

## Differences from aks-full

| Aspect | aks-full | aks-gitops |
|---|---|---|
| Helm values | In the application repo | In a separate repository |
| `config-repo` input | Not used | `myorg/my-pipeline-repo` |
| Extra secret | — | `CONFIG_REPO_TOKEN` |
| Best for | Single team, simple app | Multiple environments, platform team |

## Customization

- Replace `myorg/my-pipeline-repo` with your configuration repository
- Value file paths (`my-app/values-dev.yaml`) are relative to the config repo root
- Use `config-repo-ref` to pin a specific branch or tag from the config repo
- Add `additional-set-values` to override specific values via `--set`
