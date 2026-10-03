---
name: github-actions-pipelines
description: Current GitHub Actions practice for build/test/deploy pipelines (verified 2026-10-03) — workflow structure (CI on PR, deploy on main/tags, environments with required reviewers), OIDC to AWS/Azure/GCP instead of stored cloud keys (incl. the July 2026 immutable `sub` claim change that breaks trust policies for NEW repos), security hardening after the tj-actions compromise (SHA pinning + Dependabot, least-privilege GITHUB_TOKEN, script injection, pull_request_target, self-hosted runners, harden-runner egress), reusable workflows and composite actions, caching, matrices, concurrency, artifact attestations/provenance, current action versions, and a copy-paste baseline workflow. Use when writing or reviewing any .github/workflows file, wiring CI to a cloud, publishing packages/images, or auditing a repo before making it public. Cloud specifics live in aws-deploy, azure-deploy; infrastructure in terraform-iac; GitOps promotion in gitops-cicd-pipelines.
---

# GitHub Actions pipelines

## 1. Shape

```
pull_request  → lint, test, build, security scan, terraform plan   (no secrets for forks, no deploy)
push: main    → build once → image/artifact tagged with SHA → deploy to staging (environment: staging)
tag v*.*.* / manual approval → promote the SAME artifact to production (environment: production, required reviewers)
schedule      → dependency/vuln scan, terraform drift check
```
Rules: **build once, promote the artifact** (never rebuild for prod); tag images with the commit SHA, never deploy `latest`; PR workflows never deploy and never see prod secrets.

## 2. Baseline workflow

```yaml
name: ci
on:
  pull_request:
  push:
    branches: [main]
permissions: {}                       # default: nothing; grant per job
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

jobs:
  test:
    runs-on: ubuntu-latest
    permissions: { contents: read }
    timeout-minutes: 15
    steps:
      - uses: step-security/harden-runner@<sha> # v2.21.1 — egress audit/block
        with: { egress-policy: audit }
      - uses: actions/checkout@<sha>            # v7.0.1
        with: { persist-credentials: false }
      - uses: actions/setup-python@<sha>        # v7.0.0
        with: { python-version: "3.12", cache: pip }
      - run: pip install -r requirements.txt && pytest -q

  deploy:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    needs: test
    runs-on: ubuntu-latest
    environment: staging                      # protection rules + env-scoped secrets/vars
    permissions: { contents: read, id-token: write }   # id-token only where OIDC is used
    steps:
      - uses: actions/checkout@<sha>
      - uses: aws-actions/configure-aws-credentials@<sha>   # v6.3.0
        with:
          role-to-assume: ${{ vars.AWS_DEPLOY_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION }}
      - run: ./deploy.sh
```
Pin `<sha>` to the full commit SHA of the release with a `# vX.Y.Z` comment (Dependabot updates both).

Current releases (2026-10-03): `actions/checkout` v7.0.1 · `setup-node` v7.0.0 · `setup-python` v7.0.0 · `cache` v6.1.0 · `upload-artifact` v7.0.1 · `attest-build-provenance` v4.2.2 · `docker/build-push-action` v7.4.0 · `docker/login-action` v4.6.0 · `docker/setup-buildx-action` v4.4.1 · `aws-actions/configure-aws-credentials` v6.3.0 (node24 — needs runner ≥ v2.327.1) · `azure/login` v3.1.0 · `google-github-actions/auth` v3 · `hashicorp/setup-terraform` v4.0.1 · `opentofu/setup-opentofu` v2.0.2 · `step-security/harden-runner` v2.21.1. Re-check before copying — majors move yearly.

## 3. OIDC — no cloud keys in GitHub

GitHub issues a short-lived JWT per job (`permissions: id-token: write`); the cloud trusts `https://token.actions.githubusercontent.com` and checks the claims.

- **AWS:** IAM OIDC provider + role with trust condition on `aud = sts.amazonaws.com` and `sub`. Details → `aws-deploy`.
- **Azure:** Entra app registration or user-assigned managed identity + **federated credential** (audience `api://AzureADTokenExchange`), `azure/login` with `client-id`/`tenant-id`/`subscription-id` (IDs, not secrets). Details → `azure-deploy`.
- **GCP:** Workload Identity Federation pool + provider, **attribute condition required** (e.g. `assertion.repository_owner == 'my-org'`), prefer *direct* WIF (pool principal gets IAM roles, no service-account key or impersonation):
  ```yaml
  - uses: google-github-actions/auth@v3
    with:
      project_id: my-project
      workload_identity_provider: projects/123456789/locations/global/workloadIdentityPools/github/providers/my-repo
  ```

**Scope the trust tightly:** match `sub` to one repo **and** one branch or one environment (`repo:org/repo:environment:production`), never `repo:org/*` or `*`. Environment-scoped subjects are the cleanest: only jobs that pass the environment's protection rules can assume the prod role.

⚠️ **Immutable subject claims (since 15 Jul 2026):** repositories **created, renamed or transferred after 15 Jul 2026** get `sub` in the form
`repo:OWNER@OWNER_ID/REPO@REPO_ID:ref:refs/heads/main` (e.g. `repo:octo-org@123456/octo-repo@456789:environment:prod`). Trust policies written with the old `repo:owner/repo:...` form **won't match → AssumeRole/login fails** for new repos. Older repos keep the old format unless you opt in (org/repo OIDC settings or REST API). Get the IDs with `gh api repos/OWNER/REPO --jq '.id, .owner.id'`. Microsoft Entra federated credentials need the same migration. Prefer the immutable form for new trust policies — it stops "repo deleted and re-created by someone else" subject recycling.

## 4. Hardening checklist (do all of it)

- **Pin every third-party action to a full SHA.** Tags are mutable: in the tj-actions/changed-files compromise (Mar 2025, CVE-2025-30066, 23k+ repos) the attacker moved tags v1–v45.0.7 to a malicious commit that dumped secrets into logs. Enable the org/repo policy that **requires SHA pinning**; let Dependabot (`package-ecosystem: github-actions`) bump SHAs. (Dependabot *alerts* don't fire for SHA-pinned actions — version updates still do.)
- **Least-privilege `GITHUB_TOKEN`:** org default = read-only; top-level `permissions: {}`; grant per job (`contents: read`, `packages: write`, `id-token: write`, `pull-requests: write` only where needed). Disallow Actions from creating/approving PRs.
- **Script injection:** never put `${{ github.event.* }}` (PR titles, branch names, issue bodies, commit messages) directly in `run:`. Pass through `env:` and quote: `env: { TITLE: ${{ github.event.pull_request.title }} }` → `"$TITLE"`.
- **`pull_request_target` / `workflow_run`:** privileged context with secrets — never check out or run the PR's code in them. Use plain `pull_request` for CI on forks.
- **Fork PRs:** require approval for first-time contributors; secrets aren't passed to fork `pull_request` runs — don't work around it.
- **Self-hosted runners:** never on public repos; ephemeral/JIT runners (one job, then destroyed); runner groups restricted to specific repos; no cloud metadata access, no long-lived keys on the box.
- **`actions/checkout` with `persist-credentials: false`** unless a later step must push.
- **Secrets:** environment-scoped secrets for deploys; one value per secret (no JSON blobs — masking breaks); `::add-mask::` anything derived; delete logs that leaked a secret *and rotate it*.
- **CODEOWNERS** on `.github/workflows/` + branch protection/rulesets requiring review.
- **Egress:** `step-security/harden-runner` in audit → block mode with an allowlist catches exfiltration like tj-actions.
- **Scanning:** CodeQL default setup flags workflow injection/unpinned actions; OpenSSF Scorecard action; `zizmor` / `actionlint` locally or in CI; dependency-review-action on PRs.
- **Allowed actions policy:** org setting to allow only GitHub-owned, verified creators, or an explicit list.

## 5. Reuse without copy-paste

- **Reusable workflows** (`on: workflow_call`, `uses: org/repo/.github/workflows/deploy.yml@<sha>`) for whole pipelines; pass `secrets: inherit` only to trusted internal workflows, otherwise pass named secrets.
- **Composite actions** for repeated step groups (setup + cache + install).
- Central "platform" repo with versioned reusable workflows = one place to fix a security issue for every repo.

## 6. Speed

- Built-in caching in `setup-*` actions (`cache: npm|pip|poetry`); `actions/cache` for the rest; Docker layer cache with `cache-from/cache-to: type=gha` in build-push-action.
- `concurrency` to cancel superseded PR runs; never cancel in-progress deploys (`cancel-in-progress: false` for deploy groups).
- Path filters / change detection for monorepos — and their traps (shallow clones, shared packages) → `gitops-cicd-pipelines`.
- `timeout-minutes` on every job (default is 6 hours).
- Matrix only what matters (`fail-fast: false` when you want the full picture).

## 7. Supply chain for what you ship

- **Artifact attestations:** `actions/attest-build-provenance` (needs `id-token: write`, `attestations: write`) → SLSA provenance signed via Sigstore; consumers verify with `gh attestation verify <file|oci://image> --owner ORG`.
- **npm:** trusted publishing via OIDC (no `NPM_TOKEN`), `npm publish --provenance`.
- **Containers:** push by digest, sign/attest, generate SBOM (`docker/build-push-action` `sbom: true`, `provenance: mode=max`), scan with Trivy/Grype and fail on criticals.
- Release on **tag push** (`on: push: tags: ['v*.*.*']`), not on every main push, when consumers depend on versions.

## 8. Debugging

- `ACTIONS_STEP_DEBUG=true` (repo variable) for verbose step logs; `act` to run workflows locally (limited for OIDC/services).
- OIDC failures: print the claims you actually get (`curl "$ACTIONS_ID_TOKEN_REQUEST_URL&audience=sts.amazonaws.com" -H "Authorization: bearer $ACTIONS_ID_TOKEN_REQUEST_TOKEN"` → decode the JWT payload, **never** print the raw token in public logs) and compare `sub`/`aud` with the trust policy — 90% of "Not authorized to perform sts:AssumeRoleWithWebIdentity" is a `sub` mismatch (now often the immutable-ID format).

## Sources
- Security hardening for GitHub Actions — https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions
- OIDC in AWS / Azure — https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services · …/configuring-openid-connect-in-azure
- OIDC reference (immutable subjects) — https://docs.github.com/actions/reference/openid-connect-reference ; Entra migration — https://learn.microsoft.com/en-us/entra/workload-id/workload-identities-github-immutable-subjects
- google-github-actions/auth — https://github.com/google-github-actions/auth
- tj-actions incident — https://www.sysdig.com/blog/detecting-and-mitigating-the-tj-actions-changed-files-supply-chain-attack-cve-2025-30066
