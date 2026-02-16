# Authentication Setup

NuvTools Pipelines supports two authentication methods for Azure. OIDC is recommended as it requires no long-lived secrets.

## How OIDC Subject Claims Work

When a GitHub Actions job needs to access Azure, it requests a short-lived OIDC token from GitHub's token service. This token contains a **subject claim** — a string that identifies *what* triggered the job. Azure receives this token and checks if any federated credential on the App Registration has a matching subject. If a match is found, the job is authorized; if not, you get the `AADSTS70021: No matching federated identity record found` error.

The subject claim format depends on the trigger type:

| Trigger | Subject claim |
|---|---|
| Push to `main` | `repo:<owner>/<repo>:ref:refs/heads/main` |
| Push tag `v1.0.0` | `repo:<owner>/<repo>:ref:refs/tags/v1.0.0` |
| Pull request | `repo:<owner>/<repo>:pull_request` |
| Job with `environment: dev` | `repo:<owner>/<repo>:environment:dev` |

> **Key insight:** when a job uses the `environment:` keyword, the subject claim **changes entirely** to `repo:<owner>/<repo>:environment:<name>` — regardless of what originally triggered the workflow (branch push, tag push, etc.). This means a deploy job triggered by a tag push does **not** use a tag-based subject; it uses an environment-based subject.

This is why you typically need **two** federated credentials for a single trigger path: one for the CI job (which sees the original trigger subject) and one for the CD job (which sees the environment subject).

## Option A: OIDC Federated Credentials (Recommended)

### 1. Create App Registration

```bash
az ad app create --display-name "github-actions-<your-app>"
```

Note the `appId` from the output — this is your `AZURE_CLIENT_ID`.

### 2. Create Service Principal

```bash
az ad sp create --id <app-id>
```

### 3. Create Federated Credentials

You need one credential per subject claim. The subject format depends on the trigger type.

#### Which credentials do I need?

Use this table to determine which federated credentials your pipeline requires. Each row shows what happens when a specific trigger fires:

| Trigger | CI job subject | Deploy job subject | Credentials needed |
|---|---|---|---|
| Push to `main` | `ref:refs/heads/main` | `environment:dev` | branch-main + env-dev |
| Tag `v*-rc*` (prerelease) | `ref:refs/tags/*` | `environment:staging` | tags + env-staging |
| Tag `v*` (stable release) | `ref:refs/tags/*` | `environment:production` | tags + env-production |
| Pull request | `pull_request` | _(no deploy)_ | pr |

The CI job (build, test, Docker push) runs **without** the `environment:` keyword, so Azure sees the original trigger as the subject. The CD job (deploy) uses `environment:`, which **overrides** the subject to `environment:<name>`.

> **Note:** if your pipeline does not deploy from `main` (e.g., only tags trigger deployments), you don't need the `environment:dev` credential — but you still need the `branch-main` credential so CI can push Docker images on `main` pushes.

#### For branch pushes (dev deployments)

**When it fires:** a push to `main` triggers the CI workflow. The CI jobs (build, test, Docker push) run without `environment:`, so Azure sees the branch-based subject.

**Subject claim:** `repo:<owner>/<repo>:ref:refs/heads/main`

**Pipeline flow:**

```
push to main
  └─ ci (no environment:)                → subject: repo:…:ref:refs/heads/main  ← needs this credential
       └─ deploy-dev (environment: dev)  → subject: repo:…:environment:dev      ← needs env-dev credential
```

```bash
az ad app federated-credential create --id <app-id> --parameters '{
  "name": "github-branch-main",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:<owner>/<repo>:ref:refs/heads/main",
  "audiences": ["api://AzureADTokenExchange"],
  "description": "Deploy from main branch"
}'
```

#### For tag pushes (staging/production deployments)

**When it fires:** pushing a tag like `v1.0.0-rc.1` or `v1.0.0` triggers the CI workflow. The CI jobs (build, test, Docker push) run without `environment:`, so Azure sees the tag-based subject. The wildcard `*` matches any tag.

**Subject claim:** `repo:<owner>/<repo>:ref:refs/tags/*` (wildcard matches all `v*` tags)

**Pipeline flow:**

```
push tag v1.0.0-rc.1
  └─ ci (no environment:)                     → subject: repo:…:ref:refs/tags/v1.0.0-rc.1  ← matched by tags credential
       └─ deploy-staging (environment: staging) → subject: repo:…:environment:staging       ← needs env-staging credential

push tag v1.0.0
  └─ ci (no environment:)                          → subject: repo:…:ref:refs/tags/v1.0.0   ← matched by tags credential
       └─ deploy-production (environment: production) → subject: repo:…:environment:production ← needs env-production credential
```

Notice how the CI job and the deploy job need **different** credentials: the CI job uses the tag subject while the deploy job switches to the environment subject.

```bash
az ad app federated-credential create --id <app-id> --parameters '{
  "name": "github-tags",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:<owner>/<repo>:ref:refs/tags/*",
  "audiences": ["api://AzureADTokenExchange"],
  "description": "Deploy from tags"
}'
```

#### For GitHub Environments

**When it fires:** the CD workflows (`cd-aks.yml`, `cd-appservice.yml`) use `environment: ${{ inputs.environment }}` on the deploy job. This **overrides** the OIDC subject claim to be environment-based, regardless of the original trigger (branch push, tag push, etc.).

**Subject claim:** `repo:<owner>/<repo>:environment:<name>`

**Why you need it:** even though a tag push started the pipeline, the deploy job's `environment:` keyword makes the subject `repo:…:environment:<name>`, **not** `repo:…:ref:refs/tags/…`. Without these credentials, deployments fail even if you have the branch and tag credentials configured.

```bash
for ENV in dev staging production; do
  az ad app federated-credential create --id <app-id> --parameters "{
    \"name\": \"github-env-${ENV}\",
    \"issuer\": \"https://token.actions.githubusercontent.com\",
    \"subject\": \"repo:<owner>/<repo>:environment:${ENV}\",
    \"audiences\": [\"api://AzureADTokenExchange\"],
    \"description\": \"Deploy to ${ENV} environment\"
  }"
done
```

#### For pull requests (CI only)

**When it fires:** opening or updating a pull request targeting `main`. Only CI runs (build and test) — no deployment is triggered (`should-deploy` is `false`). Azure login is still needed if Docker build is enabled (to push the image to ACR), but since PRs don't deploy, you only need this one credential.

**Subject claim:** `repo:<owner>/<repo>:pull_request`

**Pipeline flow:**

```
open/update PR
  └─ ci (no environment:)  → subject: repo:…:pull_request  ← needs this credential
       └─ (no deploy)
```

```bash
az ad app federated-credential create --id <app-id> --parameters '{
  "name": "github-pr",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:<owner>/<repo>:pull_request",
  "audiences": ["api://AzureADTokenExchange"],
  "description": "CI for pull requests"
}'
```

#### Custom environment names

The `resolve-version` action outputs `staging` for prerelease tags (`-alpha`, `-beta`, `-rc`) and `production` for stable release tags. However, the CI workflow's `environment-override` input allows consumers to use any environment name they want (e.g., `uat` instead of `staging`).

If your consumer pipeline passes `environment: uat` to the CD workflow, the deploy job's OIDC subject becomes `repo:<owner>/<repo>:environment:uat` — **not** `environment:staging`. The federated credential must match the **actual environment name used in the workflow**, not the auto-resolved name.

```bash
# Example: if you use "uat" instead of "staging"
az ad app federated-credential create --id <app-id> --parameters '{
  "name": "github-env-uat",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:<owner>/<repo>:environment:uat",
  "audiences": ["api://AzureADTokenExchange"],
  "description": "Deploy to UAT environment"
}'
```

### Using one App Registration for multiple repositories

OIDC federated credentials include the repository name in the `subject` claim (e.g., `repo:my-org/my-repo:environment:dev`). This means each repository needs its **own set of federated credentials** — but you can add them all to the **same App Registration**. There is no need to create a new App Registration for every repo.

#### Adding a second repository

If you already have an App Registration for `repo-a` and want to onboard `repo-b`, just add new federated credentials pointing to the new repo:

```bash
APP_ID="<existing-app-id>"

for ENV in dev staging production; do
  az ad app federated-credential create --id $APP_ID --parameters "{
    \"name\": \"repo-b-env-${ENV}\",
    \"issuer\": \"https://token.actions.githubusercontent.com\",
    \"subject\": \"repo:<owner>/repo-b:environment:${ENV}\",
    \"audiences\": [\"api://AzureADTokenExchange\"],
    \"description\": \"Deploy repo-b to ${ENV}\"
  }"
done
```

Then configure `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, and `AZURE_SUBSCRIPTION_ID` in `repo-b`'s GitHub Secrets using the same values.

#### Limits and recommendations

- Azure allows up to **20 federated credentials** per App Registration. A typical repo using environments consumes 3–5 credentials (one per environment plus optional branch/PR credentials).
- For a small team with a few services, a single shared App Registration works well.
- If you approach the 20-credential limit, create a second App Registration for the next group of repos.
- If the new repository deploys to **different Azure resources** (e.g., a different AKS cluster or resource group), you will also need to add RBAC role assignments for the Service Principal on those resources — see step 4 below.

### 4. Assign RBAC Roles

```bash
SP_ID=$(az ad sp show --id <app-id> --query id -o tsv)

# ACR push (for Docker build)
az role assignment create \
  --assignee $SP_ID \
  --role AcrPush \
  --scope /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.ContainerRegistry/registries/<acr>

# AKS cluster user (for Helm deploy)
az role assignment create \
  --assignee $SP_ID \
  --role "Azure Kubernetes Service Cluster User Role" \
  --scope /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.ContainerService/managedClusters/<cluster>

# OR App Service contributor (for App Service deploy)
az role assignment create \
  --assignee $SP_ID \
  --role "Website Contributor" \
  --scope /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.Web/sites/<app>
```

### 5. Configure GitHub Secrets

| Secret | Value |
|---|---|
| `AZURE_CLIENT_ID` | App Registration client ID (`appId`) |
| `AZURE_TENANT_ID` | Azure AD tenant ID |
| `AZURE_SUBSCRIPTION_ID` | Azure subscription ID |

### 6. Workflow Permissions

The CI workflow already includes the required permissions:

```yaml
permissions:
  contents: read
  id-token: write  # Required for OIDC
```

### 7. Approval Gates

Approval gates let you require manual approval before deploying to sensitive environments like `staging` or `production`. This is configured in **GitHub Environments**, not in Azure or the workflow files.

#### Setting up approval gates

1. Go to your repository's **Settings > Environments**
2. Create environments matching the names used in your pipeline (e.g., `dev`, `staging`, `production`)
3. For environments that need approval, click on the environment and add **Required reviewers** — select the users or teams who must approve before the deploy job proceeds
4. Optionally add a **Wait timer** to introduce a delay after approval (useful for staggered rollouts)

#### How it works with the pipeline

The CD workflows (`cd-appservice.yml`, `cd-aks.yml`) already use `environment: ${{ inputs.environment }}` on the deploy job. When GitHub sees this keyword, it automatically checks whether that environment has protection rules. If reviewers are required, the job pauses and waits for approval before running.

```
push tag v1.0.0
  └─ ci (runs immediately)
       └─ deploy-production (environment: production)
            └─ ⏸ Waiting for approval from required reviewers...
                 └─ ✅ Approved → deploy runs
```

No changes are needed in your workflow files — just configure the environments in GitHub.

## Option B: Service Principal with Client Secret

Use this when OIDC is not available (e.g., GitHub Enterprise Server without OIDC support).

### 1. Create Service Principal

```bash
az ad sp create-for-rbac \
  --name "github-actions-<your-app>" \
  --role contributor \
  --scopes /subscriptions/<subscription-id>/resourceGroups/<resource-group>
```

### 2. Configure GitHub Secrets

| Secret | Value |
|---|---|
| `AZURE_CLIENT_ID` | `appId` from output |
| `AZURE_TENANT_ID` | `tenant` from output |
| `AZURE_SUBSCRIPTION_ID` | Your subscription ID |
| `AZURE_CLIENT_SECRET` | `password` from output |

### 3. How It Works

The `azure-login` action auto-detects the auth method:

- If `AZURE_CLIENT_SECRET` is empty → OIDC login
- If `AZURE_CLIENT_SECRET` is provided → Service Principal login

No changes needed in your workflow — just add or remove the secret.

## Troubleshooting

### "AADSTS70021: No matching federated identity record found"

The subject claim does not match any federated credential. Common causes:

- Missing environment credential (the CD workflows use `environment:` which changes the subject)
- Repository name mismatch (check `repo:owner/name` in the subject)
- Branch/tag pattern mismatch

### "AADSTS700024: Client assertion is not within its valid time range"

The GitHub runner's clock is skewed. This is usually transient — retry the workflow.

### "AuthorizationFailed" on az acr login / az aks get-credentials

The Service Principal lacks the required RBAC role. Check the role assignments in step 4 above.
