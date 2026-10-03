# CI/CD Toolkit — Claude Code Skills for Secure Cloud Delivery

Five skills I use to build and review delivery pipelines: GitHub Actions, AWS, Azure, Terraform/OpenTofu and GitOps. Each is a working checklist with copy-paste snippets — checked against official docs and current releases on 2026-10-03, with every workflow snippet passing `actionlint`.

| Skill | Covers |
|---|---|
| [`github-actions-pipelines`](skills/github-actions-pipelines/SKILL.md) | Pipeline shape (CI on PR, build once, promote the artifact), OIDC to AWS/Azure/GCP with no stored cloud keys — including the July 2026 immutable-subject change that breaks trust policies for new repos — hardening after the tj-actions compromise (SHA pinning, least-privilege tokens, script injection, egress control), reusable workflows, caching, attestations. |
| [`aws-deploy`](skills/aws-deploy/SKILL.md) | OIDC role setup, choosing compute (ECS Express Mode now that App Runner is closed to new customers, Fargate, Lambda, EKS, Lightsail), workflows for ECR + ECS and Lambda, Secrets Manager, rollbacks, cost traps, S3 state with native locking. |
| [`azure-deploy`](skills/azure-deploy/SKILL.md) | Federated credentials for GitHub Actions and Azure DevOps (no client secrets), Container Apps / App Service / Functions / AKS, managed-identity image pulls, Key Vault, revision and slot-based safe releases, Terraform state on Azure Storage. |
| [`terraform-iac`](skills/terraform-iac/SKILL.md) | Terraform vs OpenTofu, per-environment state, plan on PR / apply on merge with separate plan and apply roles, policy and security scanning, drift detection, refactoring with import/moved/removed blocks, secrets in state. |
| [`gitops-cicd-pipelines`](skills/gitops-cicd-pipelines/SKILL.md) | GitOps repo layout, environment promotion with Argo CD + Kargo + Helm, selective "deploy only what changed" pipelines for monorepos, OpenTelemetry wiring. |

**Use:** copy a skill folder into `~/.claude/skills/` (Claude Code) — or read them as plain Markdown checklists.

**Related:** [ai-engineering-toolkit](https://github.com/Eugene-WebDev/ai-engineering-toolkit) · [ai-security-toolkit](https://github.com/Eugene-WebDev/ai-security-toolkit)

---

*Eugene Melnychenko — AI Automation & Integrations Engineer · [linkedin.com/in/eugene-webdev](https://www.linkedin.com/in/eugene-webdev)*
