# App Service Basic (Zip Deploy)

Pipeline for deploying a .NET application to Azure App Service using artifact-based zip deploy.

## What It Demonstrates

- CI with build and test (no Docker build needed)
- CD to App Service via zip deploy (artifact mode)
- Automatic deployment to dev, staging and production
- Deployment slots with swap in production for zero-downtime

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
  .github/
    workflows/
      pipeline.yml          <-- copy from this example
```

## Prerequisites

- Azure App Service per environment (dev, staging, production)
- App Registration with OIDC federated credentials ([authentication guide](../../docs/authentication-setup.md))
- Deployment slot `staging` on the production App Service (for slot swap)
- No ACR or Docker required — this example uses artifact-based deployment

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
| `build-docker` | `false` | Docker not needed for zip deploy |

## CD Parameters

| Parameter | Description |
|---|---|
| `environment` | Target environment (`dev`, `staging`, `production`) |
| `app-version` | Application version (from CI output) |
| `deploy-mode` | `artifact` — deploy via zip package |
| `app-service-name` | App Service name for the target environment |
| `resource-group` | Azure resource group |
| `use-deployment-slot` | `true` in production for zero-downtime |
| `slot-name` | Slot name (default: `staging`) |

## Orchestration Flow

| Action | Result |
|---|---|
| Push to `main` | Build + deploy to **dev** |
| Tag `v1.0.0-rc.1` | Build + deploy to **staging** |
| Tag `v1.0.0` | Build + deploy to **production** (with slot swap) |
| Pull request | Build + test only (no deploy) |

## Customization

- To add per-environment app settings, use `app-settings-file` with a YAML key-value file
- To use an external config repo for settings, add `config-repo` and the `CONFIG_REPO_TOKEN` secret
- To switch to Docker-based deployment, see the [appservice-docker](../appservice-docker/) example
