---
name: azure-deploy
description: Deploying apps to Azure from CI (verified 2026-10-03) — GitHub Actions → Azure via OIDC (Entra app or user-assigned managed identity + federated credential, azure/login v3 with client/tenant/subscription IDs; immutable-subject migration for new repos), Azure DevOps Pipelines with workload identity federation service connections (convert secret-based ones in place), choosing a target (Container Apps, App Service, Functions, AKS), container workflows with ACR + managed-identity image pull, Key Vault for secrets, slot/revision-based safe releases, Terraform state in Azure Storage, and the gotcha that Microsoft's own Container Apps tutorial still uses a client-secret AZURE_CREDENTIALS blob. Use when wiring a repo to Azure, migrating pipelines off service-principal secrets, choosing between GitHub Actions and Azure DevOps, or debugging AADSTS federated-credential errors.
---

# Deploying to Azure from CI

## 1. GitHub Actions → Azure with OIDC

Setup:
1. Create an identity: **user-assigned managed identity** (simplest, no app registration) or an Entra **app registration + service principal**.
2. Add a **federated credential**: issuer `https://token.actions.githubusercontent.com`, audience `api://AzureADTokenExchange`, subject matching exactly, e.g. `repo:my-org/my-repo:environment:production` (or `:ref:refs/heads/main`, `:pull_request`).
   - Entra matches subjects **exactly** (no wildcards; "flexible federated identity credentials" with claim matching expressions exist for patterns). One credential per branch/environment.
   - Repos created/renamed/transferred after **15 Jul 2026** send `repo:my-org@<OWNER_ID>/my-repo@<REPO_ID>:...` — Microsoft has a migration guide for existing credentials (immutable subjects). See `github-actions-pipelines` §3.
3. Role assignment on the **narrowest scope** (resource group or the single resource), e.g. `Contributor` on the app's RG, `AcrPush` on the registry — not subscription Owner.

Workflow:
```yaml
permissions: { id-token: write, contents: read }
steps:
  - uses: azure/login@<sha>     # v3.1.0
    with:
      client-id: ${{ vars.AZURE_CLIENT_ID }}
      tenant-id: ${{ vars.AZURE_TENANT_ID }}
      subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
  - run: az account show
```
These are identifiers, not secrets (variables are fine; secrets also fine).

⚠️ Microsoft's Container Apps GitHub Actions tutorial (and many blog posts) still create `az ad sp create-for-rbac --json-auth` and store the JSON as `AZURE_CREDENTIALS` with `azure/login@v1 creds:` — that's a long-lived client secret. Replace with the OIDC login above.

## 2. Azure DevOps Pipelines

- Use **Azure Resource Manager service connections with workload identity federation** (GA) — app registration or managed identity, no secret stored.
- Existing secret-based connections can be **converted in place**; pipelines using them don't change.
- Protect prod with **Environments** (approvals and checks), restrict which pipelines may use a service connection, and keep YAML pipelines in repo (no classic release pipelines for new work).
- Key Vault: `AzureKeyVault@2` task or variable groups linked to Key Vault; mark secrets as secret variables.
- GitHub Actions vs Azure DevOps: code already on GitHub → Actions; Azure Boards/Repos/Test Plans shop or strict enterprise approvals → Azure DevOps. Both use the same federated-identity model now.

## 3. Pick the target

| Need | Service |
|---|---|
| Containers, scale-to-zero, HTTP APIs, background workers, Dapr | **Azure Container Apps** (revisions + traffic splitting) |
| Classic web app (code or container), deployment slots, easy custom domains | **App Service** (Linux) |
| Event-driven functions | **Azure Functions** (Flex Consumption plan for new apps) |
| Many services / platform team / need K8s APIs | **AKS** (+ GitOps via Flux extension or Argo CD) |
| Single VM, Docker Compose, n8n-style self-host | VM + Bastion/no public SSH; or Container Apps if it fits |

## 4. Container Apps workflow

```yaml
jobs:
  deploy:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production
    permissions: { id-token: write, contents: read }
    steps:
      - uses: actions/checkout@<sha>
      - uses: azure/login@<sha>
        with:
          client-id: ${{ vars.AZURE_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
      - uses: azure/container-apps-deploy-action@<sha>    # v2
        with:
          acrName: myregistry
          containerAppName: my-app
          resourceGroup: my-rg
          appSourcePath: ${{ github.workspace }}/src        # or imageToDeploy: myregistry.azurecr.io/app:${{ github.sha }}
```
- Image pull: give the container app a **managed identity with `AcrPull`** on the registry (`az containerapp registry set --server <acr>.azurecr.io --identity system`), not ACR admin credentials.
- Tag with the commit SHA, never `latest` (a stable tag may not create a new revision).
- Safe release: multiple-revision mode → deploy new revision with 0% traffic → smoke test its revision URL → shift traffic (`az containerapp ingress traffic set`) → deactivate old revision. Rollback = shift traffic back.
- Alternative without the action: `az acr build` + `az containerapp update --image ...`.

## 5. App Service / Functions

- `azure/webapps-deploy@<sha>` (v2.2.19) after `azure/login`; deploy to a **staging slot**, warm up, then **swap** (instant rollback by swapping back). Slot-sticky settings for env-specific config.
- Functions: `Azure/functions-action` or zip deploy via `az functionapp deployment source config-zip`; Flex Consumption for new apps.
- Use **managed identity** from the app to Key Vault, Storage, SQL (Entra auth) — no connection-string secrets where the service supports Entra.

## 6. Secrets

- **Key Vault** with RBAC authorization (not legacy access policies); apps read via managed identity; Container Apps/App Service support **Key Vault references** in app settings/secrets.
- CI needs no app secrets; the runtime identity reads them.
- Rotation, leak response → `secrets-rotation`.

## 7. Terraform state on Azure

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-tfstate"
    storage_account_name = "myorgtfstate"
    container_name       = "tfstate"
    key                  = "app/prod.tfstate"
    use_oidc             = true     # GitHub OIDC / ADO federation
    use_azuread_auth     = true     # Entra auth to the storage account, no access keys
  }
}
```
Locking uses blob leases (built in). Disable shared-key access on the state storage account, enable versioning + soft delete. Provider auth: `ARM_USE_OIDC=true`, `ARM_CLIENT_ID`, `ARM_TENANT_ID`, `ARM_SUBSCRIPTION_ID`. More → `terraform-iac`.

## 8. Ops and cost

- Application Insights / Log Analytics with an explicit retention and daily cap (ingestion is the bill).
- Container Apps scale-to-zero for dev; min replicas ≥1 for latency-sensitive prod.
- Budgets + cost alerts per subscription; tag resources with owner/env.
- Errors: `AADSTS70021: No matching federated identity record found` → subject/issuer/audience mismatch — compare the token's `sub` with the credential (often branch vs environment, or the new immutable-ID format).

## Sources
- GitHub OIDC in Azure — https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-azure
- Immutable subjects migration (Entra) — https://learn.microsoft.com/en-us/entra/workload-id/workload-identities-github-immutable-subjects
- Container Apps + GitHub Actions — https://learn.microsoft.com/en-us/azure/container-apps/github-actions
- Azure DevOps workload identity federation — https://learn.microsoft.com/en-us/azure/devops/pipelines/release/configure-workload-identity?view=azure-devops · GA post https://devblogs.microsoft.com/devops/workload-identity-federation-for-azure-deployments-is-now-generally-available/
- azure/login releases — https://github.com/Azure/login/releases
