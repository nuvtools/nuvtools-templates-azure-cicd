# Migrating from Azure DevOps

This guide maps concepts from a typical Azure DevOps GitOps pipeline template to NuvTools Pipelines on GitHub Actions.

## Concept Mapping

| Azure DevOps | GitHub Actions | Notes |
|---|---|---|
| Pipeline template repo | `nuvtools/nuvtools-templates-azure-cicd` | Reusable workflows instead of YAML templates |
| `repositoryConfiguration` | `config-repo` input | Optional — values can also be inline |
| `repositoryConsumer` | Consumer repo (triggers the workflow) | Same concept |
| Stage gates | GitHub Environments | Configure reviewers in Settings > Environments |
| Service connections | OIDC federated credentials | No shared secrets |
| Variable groups | GitHub Secrets + env-values files | Secrets at org/repo level |
| Pipeline variables | Workflow inputs | Passed via `with:` |
| Artifacts (7z) | GitHub Artifacts | Built-in compression, no 7z step needed |
| Pipeline run number | `github.run_number` | Same concept |

## Pipeline Structure Mapping

### Azure DevOps (4 stages)

```
Test → Build → Push → Deploy
```

### GitHub Actions (2 workflows)

```
CI (build+test+docker) → CD (deploy)
```

The Test and Build stages are merged into a single job for efficiency. Docker push is a separate job within CI.

## Version Detection

### Azure DevOps (scattered logic)

Version detection was spread across multiple stages and conditions:
- `Build.SourceBranch` conditions in `test-dotnet.yml`
- Tag parsing in `push.yml` and `deploy-helm-prepare.yml`
- Environment detection in each deploy stage

### GitHub Actions (centralized)

The `resolve-version` action handles all version logic in one place:
- Input: `github.ref`, `github.event_name`, `github.run_number`
- Output: `version`, `environment`, `should-deploy`

## Auth Migration

### Azure DevOps

```yaml
# Service connection reference
azureSubscription: 'my-service-connection'
```

### GitHub Actions

```yaml
# OIDC — no permanent secrets
- uses: nuvtools/nuvtools-templates-azure-cicd/.github/actions/azure-login@v1
  with:
    client-id: ${{ secrets.AZURE_CLIENT_ID }}
    tenant-id: ${{ secrets.AZURE_TENANT_ID }}
    subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

See [Authentication Setup](authentication-setup.md) for the full OIDC configuration.

## Helm Deploy Migration

### Azure DevOps

```yaml
# Previously: separate prepare + deploy stages
- task: HelmInstaller@1
- task: KubernetesManifest@0
  inputs:
    action: deploy
```

### GitHub Actions

```yaml
# All in one composite action
- uses: nuvtools/nuvtools-templates-azure-cicd/.github/actions/helm-deploy@v1
  with:
    aks-cluster-name: my-cluster
    aks-resource-group: my-rg
    namespace: my-app
    release-name: my-app
    values-file: values-dev.yaml
    image-uri: myacr.azurecr.io/my-app:dev42
    app-version: dev42
```

## Docker Push Migration

### Azure DevOps

```yaml
# push.yml — with 7z extraction
- task: ExtractFiles@1
  inputs:
    archiveFilePatterns: '*.7z'
- task: Docker@2
  inputs:
    command: buildAndPush
```

### GitHub Actions

```yaml
# No 7z — GitHub Artifacts handles compression
- uses: nuvtools/nuvtools-templates-azure-cicd/.github/actions/docker-build-push@v1
  with:
    registry: myacr.azurecr.io
    image-name: my-app
    image-tag: dev42
    entry-point-dll: MyApp.API.dll
```

## Config Repo Migration

### Azure DevOps (required)

```yaml
resources:
  repositories:
    - repository: configuration
      type: git
      name: my-project/pipeline-config
```

### GitHub Actions (optional)

```yaml
# Inline pattern (simple)
values-file: helm-values-dev.yml  # In the consumer repo

# GitOps pattern (enterprise)
config-repo: my-org/pipeline-config
values-file: my-app/values-dev.yaml
```

## Env Values Migration

### Azure DevOps

```yaml
# env-values-dev.yml processed manually
- script: |
    while read line; do
      helm upgrade --set "env.$line" ...
    done < env-values-dev.yml
```

### GitHub Actions

The `helm-deploy` action processes `env-values-file` automatically:

```yaml
- uses: nuvtools/nuvtools-templates-azure-cicd/.github/actions/helm-deploy@v1
  with:
    env-values-file: env-values-dev.yml
    # Each key:value pair becomes --set env.<key>=<value>
```

## Migration Checklist

- [ ] Create Azure AD App Registration with OIDC federated credentials
- [ ] Configure GitHub repository secrets (`AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`)
- [ ] Create GitHub Environments (dev, staging, production) with appropriate protection rules
- [ ] Copy and adapt pipeline workflow from examples
- [ ] Migrate Helm values files (replace `<IMAGE_PATH>` pattern is preserved)
- [ ] Migrate env-values files (format is unchanged)
- [ ] Remove any non-Azure cloud configuration (if present)
- [ ] Test with a push to main (dev deployment)
- [ ] Test with a prerelease tag (staging deployment)
- [ ] Test with a release tag (production deployment)
- [ ] Decommission Azure DevOps pipeline
