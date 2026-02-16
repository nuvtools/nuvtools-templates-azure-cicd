# App Service Docker

Pipeline for deploying a .NET application to Azure App Service using Docker containers.

## What It Demonstrates

- CI with build, test, Docker build and push to ACR
- CD to App Service using `docker` deploy mode (container image)
- Automatic deployment to dev, staging and production
- Deployment slots with swap in production for zero-downtime

## When to Choose Docker vs Artifact

| Criteria | Artifact (zip deploy) | Docker (container) |
|---|---|---|
| Simplicity | Simpler, no ACR needed | Requires ACR |
| Consistency | Depends on App Service runtime | Identical image across all environments |
| Startup time | Faster | May be slower (image pull) |
| Customization | Standard Azure runtime | Full control over the environment |
| Best for | Simple apps, prototyping | Microservices, custom dependencies |

Use **artifact** when you don't need control over the runtime. Use **docker** when you need consistency across environments or specific dependencies.

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
- Azure Container Registry (ACR)
- App Registration with OIDC federated credentials ([authentication guide](../../docs/authentication-setup.md))
- Deployment slot `staging` on the production App Service (for slot swap)

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
| `deploy-mode` | `docker` — deploy via container image |
| `image-uri` | Full image URI (from CI output) |
| `app-settings-file` | App settings file (e.g., `app-settings-dev.yml`) |
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
- Use `dockerfile` to point to a custom Dockerfile instead of the default
