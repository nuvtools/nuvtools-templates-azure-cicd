# Authentication Setup

NuvTools Pipelines supports two authentication methods for Azure. OIDC is recommended as it requires no long-lived secrets.

## How OIDC Subject Claims Work

When a GitHub Actions job needs to access Azure, it requests a short-lived OIDC token from GitHub's token service. This token contains a **subject claim** — a string that identifies *what* triggered the job. Azure receives this token and checks if any federated credential on the App Registration has a matching subject. If a match is found, the job is authorized; if not, you get the `AADSTS70021: No matching federated identity record found` error.

The subject claim format depends on the job:

| Job | Subject claim |
|---|---|
| CI job (no `environment:`), dispatched from `main` | `repo:<owner>/<repo>:ref:refs/heads/main` |
| Job with `environment: dev` | `repo:<owner>/<repo>:environment:dev` |

Pipelines run on manual `workflow_dispatch` from a branch (the default branch in most cases), so the CI job sees the branch-based subject for the branch you dispatch from.

> **Key insight:** when a job uses the `environment:` keyword, the subject claim **changes entirely** to `repo:<owner>/<repo>:environment:<name>` — regardless of which branch the workflow was dispatched from. The deploy job always uses an environment-based subject.

This is why you typically need **two** federated credentials: one for the CI job (which sees the branch subject of the branch you dispatch from) and one for the CD job (which sees the environment subject).

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

Use this table to determine which federated credentials your pipeline requires. The CI job runs **without** the `environment:` keyword, so Azure sees the branch you dispatch from as the subject. The CD job (deploy) uses `environment:`, which **overrides** the subject to `environment:<name>`.

| Dispatch | CI job subject | Deploy job subject | Credentials needed |
|---|---|---|---|
| `environment: dev` | `ref:refs/heads/main` | `environment:dev` | branch-main + env-dev |
| `environment: staging` | `ref:refs/heads/main` | `environment:staging` | branch-main + env-staging |
| `environment: production` | `ref:refs/heads/main` | `environment:production` | branch-main + env-production |
| `runDeploy: false` | `ref:refs/heads/main` | _(no deploy)_ | branch-main |

> **Note:** the CI job's subject is the branch you dispatch from (`main` in the examples above). If you dispatch from other branches, add a matching `branch-*` credential for each.

#### For the CI job (dispatch branch)

**When it fires:** you manually dispatch the workflow from a branch. The CI job (build, test, Docker push) runs without `environment:`, so Azure sees the branch-based subject.

**Subject claim:** `repo:<owner>/<repo>:ref:refs/heads/main`

**Pipeline flow:**

```
dispatch (environment: dev)
  └─ ci (no environment:)                → subject: repo:…:ref:refs/heads/main  ← needs this credential
       └─ deploy-dev (environment: dev)  → subject: repo:…:environment:dev      ← needs env-dev credential
```

```bash
az ad app federated-credential create --id <app-id> --parameters '{
  "name": "github-branch-main",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:<owner>/<repo>:ref:refs/heads/main",
  "audiences": ["api://AzureADTokenExchange"],
  "description": "CI dispatched from main branch"
}'
```

#### For GitHub Environments

**When it fires:** the CD workflows (`cd-aks.yml`, `cd-appservice.yml`) use `environment: ${{ inputs.environment }}` on the deploy job. This **overrides** the OIDC subject claim to be environment-based, regardless of which branch the workflow was dispatched from.

**Subject claim:** `repo:<owner>/<repo>:environment:<name>`

**Why you need it:** the deploy job's `environment:` keyword makes the subject `repo:…:environment:<name>`, **not** the branch-based subject the CI job uses. Without these credentials, deployments fail even if you have the CI branch credential configured.

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

#### Build-only dispatch (no deploy)

**When it fires:** you dispatch the workflow with `runDeploy: false`. Only CI runs (build and test) — the deploy job is skipped. Azure login is still needed if Docker build is enabled (to push the image to ACR), so the CI branch credential covers this case; no environment credential is required.

**Subject claim:** `repo:<owner>/<repo>:ref:refs/heads/main` (the branch you dispatch from)

**Pipeline flow:**

```
dispatch (runDeploy: false)
  └─ ci (no environment:)  → subject: repo:…:ref:refs/heads/main  ← covered by the branch credential
       └─ (deploy skipped)
```

#### Custom environment names

The `environment` dropdown lists whatever environment names you define in your consumer workflow (e.g., `uat` instead of `staging`).

If your consumer pipeline passes `environment: uat` to the CD workflow, the deploy job's OIDC subject becomes `repo:<owner>/<repo>:environment:uat`. The federated credential must match the **actual environment name used in the workflow**.

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

> **Tip (`cd-appservice.yml`):** these three identifiers are not secret. Instead of configuring repository secrets you can pass them as the `client-id`, `tenant-id`, and `subscription-id` workflow inputs (for example, read from a `bicepparam`). When those inputs are empty the workflow falls back to the `AZURE_*` secrets.

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
dispatch (environment: production)
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
- Branch mismatch (the CI job uses the branch you dispatch from — add a `branch-*` credential for each branch)

### "AADSTS700024: Client assertion is not within its valid time range"

The GitHub runner's clock is skewed. This is usually transient — retry the workflow.

### "AuthorizationFailed" on az acr login / az aks get-credentials

The Service Principal lacks the required RBAC role. Check the role assignments in step 4 above.
