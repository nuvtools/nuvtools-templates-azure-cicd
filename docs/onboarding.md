# Onboarding Guide

Step-by-step guide to set up NuvTools Pipelines for your .NET project.

## Prerequisites

- A .NET project hosted on GitHub
- An Azure subscription
- Azure CLI installed locally (for initial setup)
- One of:
  - **AKS**: An Azure Kubernetes Service cluster with Helm 3
  - **App Service**: An Azure App Service resource per environment

## Step 1: Create Azure AD App Registration

```bash
# Criar a App Registration
az ad app create --display-name "github-actions-my-app"

# Note the appId (CLIENT_ID) from the output
```

> **Already have an App Registration from another repo?** Skip this step and reuse its `appId` as your `AZURE_CLIENT_ID`. You only need to add new federated credentials (Step 2) and configure GitHub Secrets (Step 4) for the new repository. See [Using one App Registration for multiple repositories](authentication-setup.md#using-one-app-registration-for-multiple-repositories) for details.

## Step 2: Configure Authentication

### Option A: OIDC Federated Credentials (Recommended)

See [Authentication Setup](authentication-setup.md) for detailed instructions.

```bash
APP_ID="<your-app-id>"

# Create federated credential for the main branch
az ad app federated-credential create --id $APP_ID --parameters '{
  "name": "github-main",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:<owner>/<repo>:ref:refs/heads/main",
  "audiences": ["api://AzureADTokenExchange"]
}'

# Create federated credential for tags
az ad app federated-credential create --id $APP_ID --parameters '{
  "name": "github-tags",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:<owner>/<repo>:ref:refs/tags/*",
  "audiences": ["api://AzureADTokenExchange"]
}'

# Create federated credential for environments
for ENV in dev staging production; do
  az ad app federated-credential create --id $APP_ID --parameters "{
    \"name\": \"github-env-${ENV}\",
    \"issuer\": \"https://token.actions.githubusercontent.com\",
    \"subject\": \"repo:<owner>/<repo>:environment:${ENV}\",
    \"audiences\": [\"api://AzureADTokenExchange\"]
  }"
done
```

### Option B: Service Principal with Secret

```bash
az ad sp create-for-rbac --name "github-actions-my-app" \
  --role contributor \
  --scopes /subscriptions/<subscription-id>/resourceGroups/<resource-group> \
  --json-auth
```

Save the `clientSecret` from the output.

## Step 3: Assign Azure Permissions

```bash
SP_ID=$(az ad sp show --id $APP_ID --query id -o tsv)

# ACR permission (for Docker image push)
ACR_ID=$(az acr show --name myregistry --query id -o tsv)
az role assignment create --assignee $SP_ID --role AcrPush --scope $ACR_ID

# AKS permission (for Helm deploy)
AKS_ID=$(az aks show --name my-cluster --resource-group my-rg --query id -o tsv)
az role assignment create --assignee $SP_ID --role "Azure Kubernetes Service Cluster User Role" --scope $AKS_ID

# OR App Service permission (for zip/container deploy)
WEBAPP_ID=$(az webapp show --name my-app --resource-group my-rg --query id -o tsv)
az role assignment create --assignee $SP_ID --role "Website Contributor" --scope $WEBAPP_ID
```

## Step 4: Configure GitHub Repository

### Secrets

Go to **Settings > Secrets and variables > Actions** and add:

| Secret | Value |
|---|---|
| `AZURE_CLIENT_ID` | App Registration client ID |
| `AZURE_TENANT_ID` | Azure AD tenant ID |
| `AZURE_SUBSCRIPTION_ID` | Azure subscription ID |
| `AZURE_CLIENT_SECRET` | _(Only for SP auth)_ Client secret |

### Environments

Go to **Settings > Environments** and create:

1. **dev** — No protection rules (auto-deploy)
2. **staging** — Optional: required reviewers
3. **production** — Required reviewers, deployment branch rules (tags only)

## Step 5: Create Your Pipeline

Copy one of the examples:

- **AKS**: Copy from [`examples/aks-full/`](../examples/aks-full/)
- **App Service**: Copy from [`examples/appservice-basic/`](../examples/appservice-basic/)

Customize:
1. Update `project-path` and `test-path` to match your project
2. Set your ACR registry, image name, and entry point DLL
3. Set your cluster/App Service names and resource groups per environment
4. Create Helm values files for each environment (AKS only)

## Step 6: Push and Verify

```bash
git add .github/workflows/pipeline.yml
git commit -m "ci: add NuvTools pipeline"
git push origin main
```

Check the **Actions** tab in GitHub to verify the pipeline runs successfully.

## Next Steps

- Review the [Architecture](architecture.md) for design decisions
- Configure health check paths in your Helm values
