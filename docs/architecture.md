# Architecture

## Design Principles

1. **Convention over configuration** — Sensible defaults with optional overrides
2. **Composability** — Each action works standalone or as part of a workflow
3. **Azure-native** — Purpose-built for Azure (AKS + App Service), no multi-cloud abstractions
4. **GitOps-ready** — Optional config repo pattern for enterprise-scale deployments

## Pipeline Flow

### CI Pipeline (`ci.yml`) — Artifact-based

```
┌──────────┐     ┌─────────────┐
│ resolve   │     │ build-test  │
│ (version) │     │ (.NET)      │
└───────────┘     └─────────────┘
```

- **resolve** and **build-test** run in parallel
- No Docker job — used by App Service (zip deploy) consumers
- Outputs semantic environment names (`staging`, `production`) from `resolve-version`

### CI Pipeline (`ci-docker.yml`) — Container-based

```
┌──────────┐     ┌─────────────┐
│ resolve   │     │ build-test  │
│ (version) │     │ (.NET)      │
└─────┬─────┘     └──────┬──────┘
      │                  │
      └────────┬─────────┘
               ▼
        ┌──────────────┐
        │ docker       │
        │ (build+push) │
        └──────────────┘
```

- **resolve** and **build-test** run in parallel
- **docker** depends on both and only runs when `should-deploy: true`
- Used by AKS (Helm) and App Service (Docker) consumers
- Also outputs semantic environment names from `resolve-version`

### CD Pipeline (AKS)

```
┌──────────────────┐
│ Checkout repos   │
│ (consumer+config)│
└────────┬─────────┘
         ▼
┌──────────────────┐
│ Azure Login      │
│ (OIDC/SP)        │
└────────┬─────────┘
         ▼
┌──────────────────┐
│ Prepare chart    │
│ (substitute      │
│  placeholders)   │
└────────┬─────────┘
         ▼
┌──────────────────┐
│ Helm upgrade     │
│ --install --wait │
└────────┬─────────┘
         ▼
┌──────────────────┐
│ Verify rollout   │
└──────────────────┘
```

### CD Pipeline (App Service)

```
┌──────────────────┐
│ Azure Login      │
└────────┬─────────┘
         ▼
┌──────────────────────┐
│ Deploy               │
│ ┌──────┐ ┌────────┐  │
│ │ zip  │ │ docker │  │
│ │deploy│ │ config │  │
│ └──────┘ └────────┘  │
└────────┬─────────────┘
         ▼
┌──────────────────┐
│ Slot swap        │
│ (optional)       │
└──────────────────┘
```

## Version Resolution Strategy

The `resolve-version` action centralizes all version/environment logic:

```
git ref
  ├── refs/heads/main|master  →  dev{run_number}      → dev
  ├── refs/tags/v*-alpha*     →  semver (strip v)      → staging
  ├── refs/tags/v*-beta*      →  semver (strip v)      → staging
  ├── refs/tags/v*-rc*        →  semver (strip v)      → staging
  ├── refs/tags/v*            →  semver (strip v)      → production
  ├── refs/pull/*             →  pr{number}-{sha7}     → (none)
  └── refs/heads/*            →  {branch-slug}-{sha7}  → (none)
```

This replaces the scattered version logic that was spread across multiple stages in the Azure DevOps pipeline.

## Placeholder Substitution Pattern

A proven pattern from the original pipeline, preserved here:

### Chart.yaml
- `<APP_NAME>` → replaced with the Helm release name
- `<APP_VERSION>` → replaced with the resolved version

### Values files
- `<IMAGE_PATH>` → replaced with the container image repository (without tag)

The image tag comes from `Chart.appVersion`, so the values file only needs the repository path.

## Environment-Specific Configuration

### Inline Pattern (Simple)

Values files live in the consumer repository:

```
my-app/
├── .github/workflows/pipeline.yml
├── helm-values-dev.yml
├── helm-values-staging.yml
├── helm-values-production.yml
└── src/
```

### GitOps Config Repo Pattern (Enterprise)

Values files live in a separate configuration repository:

```
my-org/pipeline-config/
└── my-app/
    ├── values-dev.yaml
    ├── values-staging.yaml
    └── values-production.yaml
```

The consumer workflow passes `config-repo: my-org/pipeline-config` to the CD workflow, which checks out the config repo and resolves values from it.

## ConfigMap Env Vars

The `configmap-env.yaml` template renders environment variables from `Values.env` into a Kubernetes ConfigMap:

```yaml
# values.yaml
env:
  ASPNETCORE_ENVIRONMENT: Production
  DATABASE_HOST: db.example.com
```

This generates a ConfigMap referenced via `envFrom` in the Deployment, replacing hardcoded env blocks.

For sensitive values, use Kubernetes Secrets (not managed by this chart — use external-secrets or similar).

## Authentication

### OIDC (Recommended)

- No long-lived secrets stored in GitHub
- Uses GitHub's built-in OIDC token provider
- Requires federated credentials on the Azure AD app registration
- The calling workflow must set `permissions: id-token: write`

### Service Principal Fallback

- For environments where OIDC is not available
- Requires `AZURE_CLIENT_SECRET` secret
- Auto-detected: if `client-secret` input is provided, SP login is used

## Comparison with Azure DevOps Pipeline

| Aspect | Azure DevOps (old) | GitHub Actions (new) |
|---|---|---|
| Pipeline definition | YAML template + config repos | Reusable workflows + composite actions |
| Version resolution | Scattered across stages | Centralized `resolve-version` action |
| Auth | Service connections | OIDC federated credentials |
| Approval gates | Stage gates | GitHub Environments |
| Artifact transfer | 7z archive between stages | GitHub Artifacts (built-in compression) |
| Container registry | ACR | ACR |
| Orchestration | AKS | AKS |
| Config repo | Required | Optional (inline or GitOps) |
