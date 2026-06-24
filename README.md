# NuvTools Pipelines

Reusable GitHub Actions workflows and composite actions for deploying .NET applications to Azure (AKS and App Service).

## Features

| Feature | Description |
|---|---|
| **CI Pipeline** | Build, test, coverage, Docker build & push — all in one reusable workflow |
| **CD for AKS** | Helm-based deployments to Azure Kubernetes Service |
| **CD for App Service** | Zip deploy or container image deploy with optional slot swaps |
| **Manual Dispatch** | Operator picks the target environment from a `workflow_dispatch` dropdown |
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
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        type: choice
        options:
          - dev
          - staging
          - production
        default: dev
      runDeploy:
        description: 'Deploy after build (uncheck for build + test only)'
        type: boolean
        default: true

permissions:
  contents: read
  id-token: write

jobs:
  ci:
    uses: nuvtools/nuvtools-templates-azure-cicd/.github/workflows/ci-docker.yml@main
    with:
      dotnet-version: '10.0.x'
      project-path: 'src/MyApp/MyApp.csproj'
      test-path: 'tests/MyApp.Tests/MyApp.Tests.csproj'
      acr-registry: myregistry.azurecr.io
      image-name: my-app
      entry-point-dll: MyApp.dll
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

  deploy:
    needs: ci
    if: inputs.runDeploy
    uses: nuvtools/nuvtools-templates-azure-cicd/.github/workflows/cd-aks.yml@main
    with:
      environment: ${{ inputs.environment }}
      image-uri: ${{ needs.ci.outputs.image-uri }}
      app-version: ${{ needs.ci.outputs.version }}
      aks-cluster-name: my-cluster
      aks-resource-group: my-rg
      namespace: my-app
      release-name: my-app
      values-file: helm-values-${{ inputs.environment }}.yml
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

### 3. Dispatch and Deploy

Pipelines run on manual `workflow_dispatch`. From the **Actions** tab, run the workflow and pick the target environment from the dropdown:

| Dispatch input | Result |
|---|---|
| `environment: dev`, `runDeploy: true` | Build + deploy to **dev** |
| `environment: staging`, `runDeploy: true` | Build + deploy to **staging** |
| `environment: production`, `runDeploy: true` | Build + deploy to **production** |
| `runDeploy: false` | Build + test only (no deploy) |

The selected `environment` maps to a GitHub Environment, so approval gates and environment secrets apply automatically.

## Workflow Reference

### `ci.yml` — Continuous Integration (artifact)

Reusable workflow that runs build, test, coverage, and publishes build artifacts (for zip-deploy consumers). No Docker job.

<details>
<summary>Inputs</summary>

| Input | Required | Default | Description |
|---|---|---|---|
| `dotnet-version` | No | `10.0.x` | .NET SDK version |
| `project-path` | **Yes** | — | Solution or project path |
| `test-path` | No | `''` | Test project path (empty skips tests) |
| `configuration` | No | `Release` | Build configuration |
| `publish-artifacts` | No | `true` | Publish build artifacts |
| `publish-path` | No | `''` | Project to publish (defaults to `project-path`) |
| `app-version` | No | `''` | Version label for logs/summary (defaults to `${{ github.run_number }}`) |

</details>

<details>
<summary>Outputs</summary>

| Output | Description |
|---|---|
| `version` | Resolved version label |

</details>

### `ci-docker.yml` — Continuous Integration (container)

Reusable workflow that runs build, test, coverage, and builds + pushes a container image to ACR.

<details>
<summary>Inputs</summary>

| Input | Required | Default | Description |
|---|---|---|---|
| `dotnet-version` | No | `10.0.x` | .NET SDK version |
| `project-path` | **Yes** | — | Solution or project path |
| `test-path` | No | `''` | Test project path (empty skips tests) |
| `configuration` | No | `Release` | Build configuration |
| `acr-registry` | **Yes** | — | ACR registry (e.g., `myacr.azurecr.io`) |
| `image-name` | **Yes** | — | Docker image name |
| `image-tag` | No | `''` | Image tag (defaults to `${{ github.run_number }}`) |
| `dockerfile` | No | `''` | Custom Dockerfile path |
| `entry-point-dll` | No | `''` | .NET DLL entry point (default Dockerfile) |
| `publish-artifacts` | No | `true` | Publish build artifacts |
| `push` | No | `true` | Whether to push the image to the registry |

</details>

<details>
<summary>Outputs</summary>

| Output | Description |
|---|---|
| `version` | Resolved image tag |
| `image-uri` | Full Docker image URI |

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
| `app-settings-file` | No | `''` | YAML app settings file |
| `config-repo` | No | `''` | GitOps config repo |
| `config-repo-ref` | No | `main` | Config repo ref |
| `client-id` | No | `''` | Azure AD client ID for OIDC (falls back to `AZURE_CLIENT_ID` secret) |
| `tenant-id` | No | `''` | Azure AD tenant ID for OIDC (falls back to `AZURE_TENANT_ID` secret) |
| `subscription-id` | No | `''` | Azure subscription ID for OIDC (falls back to `AZURE_SUBSCRIPTION_ID` secret) |

</details>

The `client-id`, `tenant-id`, and `subscription-id` inputs are non-secret Azure OIDC identifiers. Pass them from a non-secret source (e.g. a `bicepparam`) to avoid configuring repository secrets. When left empty, the workflow falls back to the matching `AZURE_*` secrets.

## Composite Actions

Individual actions can be used standalone:

| Action | Description |
|---|---|
| `azure-login` | Azure auth with OIDC/SP auto-detection |
| `dotnet-build-test` | .NET build, test, coverage, artifacts |
| `docker-build-push` | Docker build and ACR push |
| `helm-deploy` | AKS credentials + Helm upgrade |

Usage: `uses: nuvtools/nuvtools-templates-azure-cicd/.github/actions/<action-name>@main`

## Examples

See the [examples/](examples/) directory for complete, copy-ready consumer references.

| Example | Target | Description |
|---|---|---|
| [aks-full](examples/aks-full/) | AKS | Full pipeline with per-environment Helm values in the app repo |
| [appservice-basic](examples/appservice-basic/) | App Service | Zip deploy with optional slot swap |
| [appservice-docker](examples/appservice-docker/) | App Service | Docker container deploy with slot swap |

## Documentation

- [Onboarding Guide](docs/onboarding.md) — Step-by-step setup
- [Architecture](docs/architecture.md) — Design decisions and flow diagrams
- [Authentication Setup](docs/authentication-setup.md) — OIDC and SP configuration

## License

[MIT](LICENSE)
