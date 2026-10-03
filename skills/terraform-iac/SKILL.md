---
name: terraform-iac
description: Infrastructure as code with Terraform / OpenTofu in CI (verified 2026-10-03; Terraform 1.16.x, OpenTofu 1.13.x) — choosing Terraform (BSL) vs OpenTofu (MPL, state encryption), repo layout and per-environment state, remote backends with locking (S3 use_lockfile, azurerm blob lease, GCS), plan-on-PR / apply-on-merge with the saved plan, OIDC auth to clouds, drift detection on a schedule, policy and security scanning (tflint, Trivy/Checkov, OPA/Conftest), modules and version pinning, imports and moved blocks, secrets in state, and safe destroys. Use when starting IaC for a project, wiring Terraform into GitHub Actions or Azure DevOps, reviewing a plan, fixing state/lock problems, or deciding Terraform vs OpenTofu vs click-ops.
---

# Terraform / OpenTofu in CI

## 1. Terraform or OpenTofu

| | Terraform (HashiCorp/IBM) | OpenTofu (Linux Foundation) |
|---|---|---|
| License | BSL 1.1 (fine for using it; restricts competing products) | MPL 2.0 |
| Current | 1.16.5 (2026-10-02) | 1.13.1 (2026-10-01) |
| Extras | HCP Terraform integration, Terraform Stacks | **Client-side state encryption**, early variable evaluation in backends/module sources, `-exclude` |
| Compatibility | Providers/modules shared via registry; config largely compatible — test before switching mid-project |

Default: whichever the client already uses. Greenfield and no HCP Terraform → OpenTofu is a sound choice (state encryption is a real security plus). Don't mix both on one state.

## 2. Layout

```
infra/
  modules/            # your reusable modules (versioned via git tags)
  envs/
    staging/  main.tf  backend.tf  terraform.tfvars
    prod/     main.tf  backend.tf  terraform.tfvars
```
- **Separate state per environment** (and per blast-radius area: network vs app). Directory-per-env beats workspaces for prod/staging separation (different backends, credentials, reviewers).
- Pin versions: `required_version`, `required_providers` with `~>` constraints; **commit `.terraform.lock.hcl`**.
- Modules from registries pinned to exact versions; your own via `?ref=vX.Y.Z`.

## 3. Remote state + locking

- **AWS:** S3 backend with `use_lockfile = true`, `encrypt = true` (DynamoDB locking deprecated). → `aws-deploy` §7
- **Azure:** `azurerm` backend, blob-lease locking, `use_oidc` + `use_azuread_auth`, shared keys disabled. → `azure-deploy` §7
- **GCP:** `gcs` backend (locking built in), bucket with versioning + uniform access.
- **HCP Terraform / Spacelift / env0 / Atlantis** if you want runs, policies and approvals managed for you.
- State contains secrets (DB passwords, keys, outputs) in plaintext: restrict bucket access to CI roles + break-glass admins, encrypt at rest, version it, never commit `*.tfstate`. OpenTofu can encrypt state client-side.
- Stuck lock after a crashed run: confirm no run is active, then `terraform force-unlock <LOCK_ID>`.

## 4. Pipeline: plan on PR, apply the same plan on merge

```yaml
name: terraform
on:
  pull_request: { paths: ["infra/**"] }
  push: { branches: [main], paths: ["infra/**"] }
permissions: {}
jobs:
  plan:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    environment: prod-plan                       # read-only role via OIDC
    permissions: { contents: read, id-token: write, pull-requests: write }
    defaults: { run: { working-directory: infra/envs/prod } }
    steps:
      - uses: actions/checkout@<sha>
      - uses: aws-actions/configure-aws-credentials@<sha>
        with:
          role-to-assume: ${{ vars.TF_PLAN_ROLE_ARN }}
          aws-region: eu-central-1
      - uses: hashicorp/setup-terraform@<sha>    # v4.0.1  (or opentofu/setup-opentofu v2.0.2)
      - run: terraform fmt -check -recursive
      - run: terraform init -input=false
      - run: terraform validate
      - run: terraform plan -input=false -lock-timeout=5m -out=tfplan
      - run: terraform show -no-color tfplan > plan.txt   # post as PR comment (truncate; plans can contain sensitive values)
  apply:
    if: github.event_name == 'push'
    runs-on: ubuntu-latest
    environment: prod                            # required reviewers; write role trusted only for this environment
    permissions: { contents: read, id-token: write }
    concurrency: { group: tf-prod, cancel-in-progress: false }
    defaults: { run: { working-directory: infra/envs/prod } }
    steps:
      - uses: actions/checkout@<sha>
      - uses: aws-actions/configure-aws-credentials@<sha>
        with:
          role-to-assume: ${{ vars.TF_APPLY_ROLE_ARN }}
          aws-region: eu-central-1
      - uses: hashicorp/setup-terraform@<sha>
      - run: terraform init -input=false
      - run: terraform plan -input=false -out=tfplan && terraform apply -input=false tfplan
```
- **Two roles:** plan role read-only (+ state read/lock), apply role with write — trusted only for the `prod` environment subject.
- Strictest variant: upload `tfplan` from the reviewed PR (or from a manual "plan" job on main) as an artifact and apply exactly that file after approval; re-plan if the state changed.
- `concurrency` with `cancel-in-progress: false` — never kill an apply mid-way.
- Never `-auto-approve` on prod without an approval gate in front.

## 5. Checks to run on every PR

- `terraform fmt -check`, `validate`, **tflint** (provider rules: invalid instance types, deprecated args).
- **Security/misconfig:** Trivy (`trivy config .` — absorbed tfsec) or Checkov; fail on HIGH/CRITICAL, allow documented suppressions.
- **Policy as code:** OPA/Conftest on `terraform show -json tfplan` (e.g. "no public S3", "tags required", "no 0.0.0.0/0 on 22"), or Sentinel/OPA in HCP Terraform.
- **Cost:** Infracost comment on the PR for anything that adds resources.
- Review the plan for **`-/+` replace** and **destroy** lines — the dangerous ones hide in big diffs. Add `lifecycle { prevent_destroy = true }` on databases, state buckets, KMS keys.

## 6. Drift detection

Scheduled workflow (daily/weekly): `terraform plan -detailed-exitcode` → exit 2 = drift → open an issue / Slack alert. Fix by either importing the manual change into code or re-applying to revert it. Click-ops in prod is the usual cause; restrict console write access.

## 7. Refactoring without destroying

- `import` blocks (declarative imports in code) to adopt existing resources; `terraform plan -generate-config-out=generated.tf` to draft config.
- `moved` blocks when renaming resources/modules (prevents destroy+create).
- `removed` blocks to stop managing a resource without destroying it.
- `terraform state` commands only as a last resort, with a state backup first.

## 8. Secrets

- Don't put secrets in `.tfvars` in git. Read from the secrets manager via data sources, or generate them in Terraform (`random_password`) and write them straight into Secrets Manager / Key Vault — accepting they also live in state (protect state accordingly).
- Mark variables/outputs `sensitive = true` (hides from CLI output, not from state). Newer Terraform supports **ephemeral** values/resources and write-only arguments that never persist to state — prefer them where providers support them.
- Provider credentials: OIDC from CI (`ARM_USE_OIDC`, AWS role, GCP WIF), never static keys in variables.

## 9. When not to

Tiny single-VM projects with no repeatable environments can live with a documented script + `docker compose` — but anything with more than one environment, a database, DNS and IAM pays back IaC within weeks (and makes disaster recovery a `terraform apply`).

## Sources
- Terraform releases — https://github.com/hashicorp/terraform/releases · OpenTofu releases — https://github.com/opentofu/opentofu/releases
- S3 native locking / DynamoDB deprecation — https://github.com/hashicorp/terraform/pull/36257
- setup-terraform — https://github.com/hashicorp/setup-terraform · setup-opentofu — https://github.com/opentofu/setup-opentofu
- Azure Terraform OIDC (GitHub Actions + Azure DevOps) — https://techcommunity.microsoft.com/blog/azureinfrastructureblog/modernizing-terraform-pipelines-on-azure-oidc-federation-for-github-actions-and-/4516620
