# NuvTools Pipelines

Reusable GitHub Actions workflows and composite actions for deploying .NET applications to Azure (AKS and App Service).

## Features

| Feature | Description |
|---|---|
| **CI Pipeline** | Build, test, coverage, Docker build & push — all in one reusable workflow |
| **CD for AKS** | Helm-based deployments to Azure Kubernetes Service |
| **CD for App Service** | Zip deploy or container image deploy with optional slot swaps |
| **Version Resolution** | Automatic environment detection from git refs (branches, tags, PRs) |
| **Azure Auth** | OIDC (recommended) with Service Principal fallback |
| **GitOps Support** | Optional config repo pattern for Helm values per environment |
| **Default Helm Chart** | Generic chart with ConfigMap env var injection, health checks, HPA |
| **Default Dockerfile** | .NET runtime Dockerfile ready for port 8080 (non-root) |

## Quickstart

### 1. Configure Azure Authentication

Set up OIDC federated credentials (see [Authentication Setup](docs/authentication-setup.md)) and add these repository secrets:

- `AZURE_CLIENT_ID`
- `AZURE_TENANT_ID`
- `AZURE_SUBSCRIPTION_ID`

### 2. Create Your Pipeline

```yaml
# .github/workflows/pipeline.yml
name: Pipeline

on:
  push:
    branches: [main]
    tags: ['v*']
  pull_request:
    branches: [main]

jobs:
  ci:
    uses: nuvtools/nuvtools-templates-azure-cicd/.github/workflows/ci.yml@v1
    with:
      dotnet-version: '10.0.x'
      project-path: 'src/MyApp/MyApp.csproj'
      test-path: 'tests/MyApp.Tests/MyApp.Tests.csproj'
      build-docker: true
      acr-registry: myregistry.azurecr.io
      image-name: my-app
      entry-point-dll: MyApp.dll
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

  deploy:
    needs: ci
    if: needs.ci.outputs.should-deploy == 'true'
    uses: nuvtools/nuvtools-templates-azure-cicd/.github/workflows/cd-aks.yml@v1
    with:
      environment: ${{ needs.ci.outputs.environment }}
      image-uri: ${{ needs.ci.outputs.image-uri }}
      app-version: ${{ needs.ci.outputs.version }}
      aks-cluster-name: my-cluster
      aks-resource-group: my-rg
      namespace: my-app
      release-name: my-app
      values-file: helm-values-${{ needs.ci.outputs.environment }}.yml
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

### 3. Push and Deploy

| Action | Result |
|---|---|
| Push to `main` | Build + deploy to **dev** |
| Tag `v1.0.0-rc.1` | Build + deploy to **staging** |
| Tag `v1.0.0` | Build + deploy to **production** |
| Pull request | Build + test only (no deploy) |

## Version Resolution

The `resolve-version` action maps git events to versions and environments:

| Git Event | Image Tag | Environment |
|---|---|---|
| Push to `main`/`master` | `dev{run_number}` | `dev` |
| Tag `v*-alpha*`/`v*-beta*`/`v*-rc*` | SemVer (strip `v`) | `staging` |
| Tag `v*` (stable) | SemVer (strip `v`) | `production` |
| Pull request | `pr{number}-{sha7}` | _(none — CI only)_ |

## Workflow Reference

### `ci.yml` — Continuous Integration

Reusable workflow that runs build, test, and optionally Docker build + push.

<details>
<summary>Inputs</summary>

| Input | Required | Default | Description |
|---|---|---|---|
| `dotnet-version` | No | `10.0.x` | .NET SDK version |
| `project-path` | **Yes** | — | Solution or project path |
| `test-path` | No | `''` | Test project path |
| `configuration` | No | `Release` | Build configuration |
| `build-docker` | No | `true` | Build and push Docker image |
| `acr-registry` | No | `''` | ACR registry (e.g., `myacr.azurecr.io`) |
| `image-name` | No | `''` | Docker image name |
| `dockerfile` | No | `''` | Custom Dockerfile path |
| `entry-point-dll` | No | `''` | .NET DLL entry point |
| `environment-override` | No | `''` | Override auto-detected environment |

</details>

<details>
<summary>Outputs</summary>

| Output | Description |
|---|---|
| `version` | Resolved version/tag |
| `environment` | Target environment |
| `image-uri` | Full Docker image URI |
| `should-deploy` | Whether to deploy |

</details>

### `cd-aks.yml` — Deploy to AKS

Deploys to Azure Kubernetes Service via Helm.

<details>
<summary>Inputs</summary>

| Input | Required | Default | Description |
|---|---|---|---|
| `environment` | **Yes** | — | Target environment |
| `image-uri` | **Yes** | — | Container image URI |
| `app-version` | **Yes** | — | App version |
| `aks-cluster-name` | **Yes** | — | AKS cluster name |
| `aks-resource-group` | **Yes** | — | Azure resource group |
| `namespace` | **Yes** | — | K8s namespace |
| `release-name` | **Yes** | — | Helm release name |
| `values-file` | **Yes** | — | Helm values file path |
| `chart-path` | No | `''` | Custom chart (default: built-in) |
| `env-values-file` | No | `''` | Env vars YAML file |
| `helm-timeout` | No | `5m0s` | Helm timeout |
| `additional-set-values` | No | `''` | Extra `--set` args |
| `config-repo` | No | `''` | GitOps config repo |
| `config-repo-ref` | No | `main` | Config repo ref |

</details>

### `cd-appservice.yml` — Deploy to App Service

Deploys to Azure App Service via zip deploy or container image.

<details>
<summary>Inputs</summary>

| Input | Required | Default | Description |
|---|---|---|---|
| `environment` | **Yes** | — | Target environment |
| `app-version` | **Yes** | — | App version |
| `deploy-mode` | No | `artifact` | `artifact` (zip) or `docker` |
| `app-service-name` | **Yes** | — | App Service name |
| `resource-group` | **Yes** | — | Azure resource group |
| `use-deployment-slot` | No | `false` | Deploy to slot first, then swap |
| `slot-name` | No | `staging` | Deployment slot name |
| `artifact-name` | No | `build-output` | Build artifact name |
| `image-uri` | No | `''` | Container image URI |
| `app-settings-file` | No | `''` | JSON app settings file |
| `config-repo` | No | `''` | GitOps config repo |
| `config-repo-ref` | No | `main` | Config repo ref |

</details>

## Composite Actions

Individual actions can be used standalone:

| Action | Description |
|---|---|
| `azure-login` | Azure auth with OIDC/SP auto-detection |
| `resolve-version` | Version and environment resolution |
| `dotnet-build-test` | .NET build, test, coverage, artifacts |
| `docker-build-push` | Docker build and ACR push |
| `helm-deploy` | AKS credentials + Helm upgrade |

Usage: `uses: nuvtools/nuvtools-templates-azure-cicd/.github/actions/<action-name>@v1`

## Examples

See the [examples/](examples/) directory for complete, copy-ready consumer references.

| Example | Target | Description |
|---|---|---|
| [aks-full](examples/aks-full/) | AKS | Full pipeline with per-environment Helm values in the app repo |
| [aks-gitops](examples/aks-gitops/) | AKS | Pipeline using an external GitOps config repository |
| [appservice-basic](examples/appservice-basic/) | App Service | Zip deploy with optional slot swap |
| [appservice-docker](examples/appservice-docker/) | App Service | Docker container deploy with slot swap |
| [pipeline-repo](examples/pipeline-repo/) | — | GitOps configuration repository structure |

## Documentation

- [Onboarding Guide](docs/onboarding.md) — Step-by-step setup
- [Architecture](docs/architecture.md) — Design decisions and flow diagrams
- [Authentication Setup](docs/authentication-setup.md) — OIDC and SP configuration
- [Migrating from Azure DevOps](docs/migrating-from-azure-devops.md) — Concept mapping

## License

[MIT](LICENSE)
