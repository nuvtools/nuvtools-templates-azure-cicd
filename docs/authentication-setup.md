# Authentication Setup

NuvTools Pipelines supports two authentication methods for Azure. OIDC is recommended as it requires no long-lived secrets.

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

#### For branch pushes (dev deployments)

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

When using the CD workflows (`cd-aks.yml`, `cd-appservice.yml`), the `environment:` keyword changes the subject claim. You need credentials for each environment:

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

```bash
az ad app federated-credential create --id <app-id> --parameters '{
  "name": "github-pr",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:<owner>/<repo>:pull_request",
  "audiences": ["api://AzureADTokenExchange"],
  "description": "CI for pull requests"
}'
```

### 4. Assign RBAC Roles

```bash
SP_ID=$(az ad sp show --id <app-id> --query id -o tsv)

# ACR push (para build Docker)
az role assignment create \
  --assignee $SP_ID \
  --role AcrPush \
  --scope /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.ContainerRegistry/registries/<acr>

# AKS cluster user (para deploy Helm)
az role assignment create \
  --assignee $SP_ID \
  --role "Azure Kubernetes Service Cluster User Role" \
  --scope /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.ContainerService/managedClusters/<cluster>

# OU App Service contributor (para deploy App Service)
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
  id-token: write  # Necessário para OIDC
```

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
