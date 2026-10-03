---
name: aws-deploy
description: Deploying apps to AWS from CI (verified 2026-10-03) — GitHub Actions → AWS via OIDC (IAM OIDC provider, role trust policy scoped by sub/environment, immutable-subject format for new repos), choosing a compute target (ECS Express Mode — App Runner closed to new customers since 30 Apr 2026; ECS on Fargate; Lambda; EKS; Lightsail/EC2 for simple VPS-style), copy-paste workflows for ECR + ECS Express Mode, ECS task-definition deploys, and Lambda (aws-actions/aws-lambda-deploy), secrets via Secrets Manager/SSM Parameter Store, least-privilege deploy roles, rollbacks, cost traps, and Terraform state on S3 with native locking. Use when wiring a repo to deploy on AWS, migrating off App Runner or long-lived IAM user keys, debugging AssumeRoleWithWebIdentity errors, or picking an AWS service for a container/API/n8n/agent backend.
---

# Deploying to AWS from CI

## 1. Auth: OIDC role, no IAM user keys

One-time setup (Terraform or console):
1. IAM → Identity providers → OpenID Connect: URL `https://token.actions.githubusercontent.com`, audience `sts.amazonaws.com`.
2. Role per repo × environment, trust policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com" },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
        "token.actions.githubusercontent.com:sub": "repo:my-org/my-repo:environment:production"
      }
    }
  }]
}
```
   Repos created/renamed/transferred **after 15 Jul 2026** send `repo:my-org@<OWNER_ID>/my-repo@<REPO_ID>:environment:production` — write the trust policy in that form (IDs: `gh api repos/my-org/my-repo --jq '.id, .owner.id'`). See `github-actions-pipelines` §3.
3. Permissions policy on the role: only what the deploy needs (push to one ECR repo, update one service, pass the specific task roles) — not `AdministratorAccess`.

Workflow side:
```yaml
permissions: { id-token: write, contents: read }
steps:
  - uses: aws-actions/configure-aws-credentials@<sha>   # v6.3.0
    with:
      role-to-assume: arn:aws:iam::<ACCOUNT_ID>:role/gha-deploy-prod
      role-session-name: gha-${{ github.run_id }}
      aws-region: eu-central-1
```
Separate AWS accounts for staging and prod (AWS Organizations) beat separate roles in one account.

## 2. Pick the compute

| Need | Service |
|---|---|
| Containerized web app/API, minimal ops | **ECS Express Mode** (image + 2 roles → ALB, HTTPS URL, autoscaling, security groups created for you). AWS's stated replacement for App Runner. |
| Containers with full control (sidecars, private networking, workers) | **ECS on Fargate** with task definitions |
| Event-driven / spiky / small APIs | **Lambda** (zip or container image) + API Gateway or function URL |
| Already on Kubernetes / many services / portability | **EKS** (+ GitOps with Argo CD → `gitops-cicd-pipelines`) |
| Single VM, n8n/Postgres/Docker Compose, fixed cheap price | **Lightsail** or EC2 (+ SSM Session Manager, no open SSH) |
| ❌ New App Runner services | Closed to new customers since **30 Apr 2026** (maintenance mode; existing services keep running, no new features) |

## 3. ECR + ECS Express Mode (from the AWS sample)

```yaml
jobs:
  deploy:
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'   # the AWS sample also deploys on PRs — don't
    runs-on: ubuntu-latest
    environment: production
    permissions: { id-token: write, contents: read }
    steps:
      - uses: actions/checkout@<sha>
      - uses: aws-actions/configure-aws-credentials@<sha>
        with:
          aws-region: ${{ vars.AWS_REGION }}
          role-to-assume: ${{ vars.AWS_DEPLOY_ROLE_ARN }}
      - id: ecr
        uses: aws-actions/amazon-ecr-login@<sha>                 # v2.1.7
      - uses: docker/build-push-action@<sha>                     # v7.4.0
        with:
          push: true
          tags: ${{ steps.ecr.outputs.registry }}/${{ vars.ECR_REPOSITORY }}:${{ github.sha }}
      - uses: aws-actions/amazon-ecs-deploy-express-service@<sha>   # v1.2.2
        with:
          service-name: ${{ vars.ECS_SERVICE }}
          cluster: ${{ vars.ECS_CLUSTER }}
          image: ${{ steps.ecr.outputs.registry }}/${{ vars.ECR_REPOSITORY }}:${{ github.sha }}
          container-port: 8080
          execution-role-arn: arn:aws:iam::${{ vars.AWS_ACCOUNT_ID }}:role/ecsTaskExecutionRole
          task-role-arn: arn:aws:iam::${{ vars.AWS_ACCOUNT_ID }}:role/my-app-task-role      # separate from execution role
          infrastructure-role-arn: arn:aws:iam::${{ vars.AWS_ACCOUNT_ID }}:role/ecsInfrastructureRoleForExpressServices
```
CLI equivalent: `aws ecs create-express-gateway-service` / `describe-express-gateway-service`. Defaults: 1 vCPU / 2 GB, internet-facing ALB in the default VPC, CPU autoscaling — override for production networking.

**Execution role** (pull image, read secrets, write logs) ≠ **task role** (what the app itself may call). The AWS sample reuses the execution role as task role; give the app its own minimal task role.

## 4. Classic ECS (task definitions)

`amazon-ecs-render-task-definition@v1.9.0` (inject image into `task-definition.json`) → `amazon-ecs-deploy-task-definition@v2.6.3` with `wait-for-service-stability: true`. Enable the **deployment circuit breaker with rollback** on the service so a failing deployment rolls back automatically. Blue/green: ECS native blue/green deployments (or CodeDeploy) with test listener + bake time.

## 5. Lambda

```yaml
- uses: aws-actions/aws-lambda-deploy@<sha>    # v1.1.2 (official, Aug 2025)
  with:
    function-name: my-function
    code-artifacts-dir: dist
    handler: index.handler
    runtime: nodejs22.x
    # container image: package-type: Image, image-uri: <ecr-uri>:<sha>
    # first deploy needs role: <execution role ARN>; large zips: s3-bucket; dry-run: true to validate
```
Publish versions + an alias (`live`) and shift traffic gradually (CodeDeploy linear/canary) for risky changes. For multi-function apps use SAM/CDK/Terraform instead of one action per function.

## 6. Secrets and config

- App secrets in **Secrets Manager** (rotation built in for RDS etc.) or **SSM Parameter Store SecureString** (cheaper, no rotation); ECS injects them via `secrets:` in the task definition (`valueFrom` ARN) — never as plaintext env in the task def or image.
- CI holds **no app secrets** — the runtime role reads them.
- Lambda: Parameters and Secrets extension or SDK at cold start with caching.
- Rotation and leak response → `secrets-rotation`.

## 7. Terraform state on AWS

```hcl
terraform {
  backend "s3" {
    bucket       = "my-org-tfstate"
    key          = "app/prod/terraform.tfstate"
    region       = "eu-central-1"
    encrypt      = true
    use_lockfile = true        # S3-native locking (TF ≥ 1.10, default path since 1.11) — no DynamoDB table
  }
}
```
`dynamodb_table` locking is deprecated. Versioned + encrypted bucket, block public access, separate state per environment. More → `terraform-iac`.

## 8. Rollback, observability, cost

- Rollback = redeploy the previous image SHA (keep ECR lifecycle rules generous enough) or let the circuit breaker do it.
- CloudWatch Logs retention set explicitly (default is forever = growing bill); alarms on 5xx, latency, task restarts.
- Cost traps: NAT Gateway (~$30+/month each + data) — use VPC endpoints for ECR/S3/Secrets Manager or public subnets for small apps; idle ALBs; unbounded log retention; forgotten EIPs; cross-AZ data transfer. Set AWS Budgets alerts on day one.

## Sources
- GitHub OIDC in AWS — https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services
- App Runner availability change — https://docs.aws.amazon.com/apprunner/latest/dg/apprunner-availability-change.html
- ECS Express Mode — https://docs.aws.amazon.com/AmazonECS/latest/developerguide/express-service-getting-started.html · sample https://github.com/aws-samples/sample-amazon-ecs-express-github-actions
- aws-lambda-deploy — https://github.com/aws-actions/aws-lambda-deploy
- Terraform S3 native locking — https://github.com/hashicorp/terraform/pull/36257
