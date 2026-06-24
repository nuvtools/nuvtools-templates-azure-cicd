# App Service Basic (Zip Deploy)

Pipeline for deploying a .NET application to Azure App Service using artifact-based zip deploy.

## What It Demonstrates

- CI with build and test (no Docker build needed)
- CD to App Service via zip deploy (artifact mode)
- Per-environment app settings applied before each deployment
- Manual dispatch to dev, staging or production via the `environment` dropdown
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
  app-settings-dev.yml              (optional - per-environment app settings)
  app-settings-staging.yml
  app-settings-production.yml
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

This example calls `ci.yml` (artifact-based, no Docker).

## CD Parameters

| Parameter | Description |
|---|---|
| `environment` | Target environment (`dev`, `staging`, `production`) |
| `app-version` | Application version (from CI output) |
| `deploy-mode` | `artifact` — deploy via zip package |
| `app-service-name` | App Service name for the target environment |
| `resource-group` | Azure resource group |
| `app-settings-file` | App settings file (e.g., `app-settings-dev.yml`) |
| `use-deployment-slot` | `true` in production for zero-downtime |
| `slot-name` | Slot name (default: `staging`) |

## Orchestration Flow

Run the workflow manually from the **Actions** tab and pick the target environment:

| Dispatch input | Result |
|---|---|
| `environment: dev`, `runDeploy: true` | Build + deploy to **dev** |
| `environment: staging`, `runDeploy: true` | Build + deploy to **staging** |
| `environment: production`, `runDeploy: true` | Build + deploy to **production** (with slot swap) |
| `runDeploy: false` | Build + test only (no deploy) |

## Customization

- To switch to Docker-based deployment, see the [appservice-docker](../appservice-docker/) example
