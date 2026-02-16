# AKS Full (In-Repo Values)

Complete pipeline for deploying a .NET application to AKS with per-environment Helm values stored in the application repository.

## What It Demonstrates

- CI with build, test, Docker build and push to ACR
- CD to AKS via Helm with per-environment values files
- Automatic deployment to dev, staging and production based on git refs
- Per-environment Helm values and env-values for ConfigMap injection

## Consumer Repository Structure

```
my-app/
  src/
    MyApp.API/
      MyApp.API.csproj
      Program.cs
  tests/
    MyApp.UnitTests/
      MyApp.UnitTests.csproj
  helm-values-dev.yml
  helm-values-staging.yml
  helm-values-production.yml
  env-values-dev.yml             (optional - env vars for ConfigMap)
  env-values-staging.yml
  env-values-production.yml
  .github/
    workflows/
      pipeline.yml               <-- copy from this example
```

## Prerequisites

- AKS cluster per environment (dev, staging, production)
- Azure Container Registry (ACR)
- App Registration with OIDC federated credentials ([authentication guide](../../docs/authentication-setup.md))

## Required Secrets

| Secret | Description |
|---|---|
| `AZURE_CLIENT_ID` | App Registration Client ID |
| `AZURE_TENANT_ID` | Azure AD Tenant ID |
| `AZURE_SUBSCRIPTION_ID` | Azure Subscription ID |

## CI Parameters

| Parameter | Value | Description |
|---|---|---|
| `dotnet-version` | `10.0.x` | .NET SDK version |
| `project-path` | `src/MyApp.API/MyApp.API.csproj` | Project path |
| `test-path` | `tests/MyApp.UnitTests/MyApp.UnitTests.csproj` | Test project path |
| `build-docker` | `true` | Enable Docker image build |
| `acr-registry` | `myregistry.azurecr.io` | ACR registry |
| `image-name` | `my-app-api` | Image name |
| `entry-point-dll` | `MyApp.API.dll` | Entry point DLL |

## CD Parameters

| Parameter | Description |
|---|---|
| `environment` | Target environment (`dev`, `staging`, `production`) |
| `image-uri` | Full image URI (from CI output) |
| `app-version` | Application version (from CI output) |
| `aks-cluster-name` | AKS cluster name for the target environment |
| `aks-resource-group` | Azure resource group containing the AKS cluster |
| `namespace` | Kubernetes namespace |
| `release-name` | Helm release name |
| `values-file` | Helm values file (e.g., `helm-values-dev.yml`) |
| `env-values-file` | Env vars file for ConfigMap (e.g., `env-values-dev.yml`) |

## Orchestration Flow

| Action | Result |
|---|---|
| Push to `main` | Build + deploy to **dev** |
| Tag `v1.0.0-rc.1` | Build + deploy to **staging** |
| Tag `v1.0.0` | Build + deploy to **production** |
| Pull request | Build + test only (no deploy) |

## Customization

- Add `chart-path` to use a custom Helm chart instead of the built-in default
- Add `additional-set-values` to override specific Helm values via `--set`
- Add `helm-timeout` to customize the Helm upgrade timeout (default: `5m0s`)
