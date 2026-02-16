# Pipeline Repo (GitOps Config Repository)

Example structure for a GitOps configuration repository that stores Helm values and environment settings separately from the application code.

## What It Demonstrates

- Per-environment Helm values (`values-dev.yaml`, `values-staging.yaml`, `values-production.yaml`)
- Centralized configuration management for one or more applications
- Separation of deployment configuration from application code

## When to Use a Config Repo

| Scenario | Recommendation |
|---|---|
| Single team, single app | Keep values in the app repo ([aks-full](../aks-full/)) |
| Platform team managing multiple apps | Use a config repo (this pattern) |
| Strict change control on production config | Use a config repo with branch protection |
| Config changes without app rebuild | Use a config repo |

## Repository Structure

```
my-pipeline-repo/
  my-app/
    values-dev.yaml
    values-staging.yaml
    values-production.yaml
  another-app/
    values-dev.yaml
    values-staging.yaml
    values-production.yaml
```

Each application has its own directory with per-environment values files.

## How Consumer Pipelines Reference It

In the consumer pipeline, set the `config-repo` input and provide the `CONFIG_REPO_TOKEN` secret:

```yaml
deploy-dev:
  uses: nuvtools/nuvtools-templates-azure-cicd/.github/workflows/cd-aks.yml@v1
  with:
    # ... other inputs ...
    values-file: my-app/values-dev.yaml        # path relative to config repo root
    config-repo: myorg/my-pipeline-repo        # owner/repo
  secrets:
    # ... Azure secrets ...
    CONFIG_REPO_TOKEN: ${{ secrets.CONFIG_REPO_TOKEN }}
```

The `cd-aks.yml` workflow checks out the config repo and resolves value file paths automatically.

## Values File Format

Values files follow standard Helm values format. Placeholders `<IMAGE_PATH>` and `<APP_VERSION>` are automatically replaced at deploy time:

```yaml
replicaCount: 1

image:
  repository: <IMAGE_PATH>
  pullPolicy: Always

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

env:
  ASPNETCORE_ENVIRONMENT: Development
  LOG_LEVEL: Debug
```

## Related Examples

- [aks-gitops](../aks-gitops/) — Complete pipeline that uses this config repo pattern
- [aks-full](../aks-full/) — Alternative approach with values in the app repo
