# CI/CD Pipeline

All deployments use **GitHub Actions with OIDC** — no long-lived AWS credentials anywhere. Each repository has a dedicated IAM role with least-privilege permissions scoped to what that repo deploys.

---

## Repository Structure

The platform is split across four GitHub repositories under the `analyticshealth` org:

| Repository | IaC Tool | Deploys | IAM Role |
|---|---|---|---|
| `analyticshealth-infra` | AWS CDK (TypeScript) | All CDK stacks | `analyticshealth-infra-deploy` |
| `analyticshealth-ingestion` | AWS SAM | Ingestion Lambdas | `analyticshealth-ingestion-deploy` |
| `analyticshealth-api` | Lambda ZIP deploy | API Lambdas | `analyticshealth-api-deploy` |
| `analyticshealth-ai` | Lambda ZIP deploy | AI/chat Lambdas | `analyticshealth-ai-deploy` |

The infra repo is the dependency root — it must be deployed first because it exports the IAM role ARNs and CloudFormation outputs consumed by all other repos.

---

## Branch Strategy

Both `infra` and `ingestion` repos follow the same two-workflow pattern:

```
feature/* branch  →  diff/changeset preview (no deploy)
main branch       →  deploy with production environment gate
```

### Feature Branch (safe preview)

```
push feature/* → Unit Tests → SAM Diff / CDK Diff
                               (--no-execute-changeset)
```

The diff job runs `sam deploy --no-execute-changeset` (ingestion) or `cdk diff` (infra). This creates and displays the changeset without executing it — a free dry-run that catches IAM and resource changes before merging.

### Main Branch (deploy)

```
push main → Unit Tests → SAM Deploy / CDK Deploy
                         environment: production
                         (requires manual approval in GitHub Environments)
```

The `environment: production` setting enables a **manual approval gate** in GitHub Environments. No deployment reaches AWS without an explicit approval click.

---

## OIDC Authentication

GitHub Actions assumes an AWS IAM role directly using OpenID Connect — no static access keys are stored in GitHub Secrets.

```yaml
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: ${{ secrets.AWS_INGESTION_DEPLOY_ROLE_ARN }}
    aws-region: eu-west-1
    audience: sts.amazonaws.com
```

**How it works:**

1. GitHub generates a short-lived OIDC token signed by `token.actions.githubusercontent.com`
2. The token contains the repository and branch as claims (`sub` and `aud`)
3. AWS STS validates the token against the OIDC provider (pre-registered in `oidc_stack.ts`)
4. AWS issues temporary credentials scoped to the IAM role
5. Credentials are valid for 1 hour, then automatically expire

The trust policy on each role restricts assumption to the exact repository:

```json
{
  "StringLike": {
    "token.actions.githubusercontent.com:sub": "repo:analyticshealth/analyticshealth-ingestion:*"
  },
  "StringEquals": {
    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
  }
}
```

The `*` at the end allows any branch and event type within that repository. A `ref:refs/heads/main` suffix would restrict to main-only — a viable hardening step for prod.

---

## IAM Role Permissions

Each role has the minimum permissions to do its job. The roles are defined in `lib/oidc_stack.ts` and all string values (role names, ARN prefixes, repo names) are sourced from `bin/constants.ts` — see [Constants Pattern](constants.md).

### `analyticshealth-infra-deploy`
- `PowerUserAccess` (managed policy) — required for CDK bootstrap and all stack operations
- `iam:*` on `cdk-*` roles — CDK needs to create/update its own bootstrap roles

### `analyticshealth-ingestion-deploy`
Scoped to SAM deploy operations only:

| Permission | Scope | Why |
|---|---|---|
| `cloudformation:*` changeset actions | `analyticshealth-ingestion-*` and `aws-sam-cli-managed-default/*` stacks | SAM manages both stacks |
| `cloudformation:CreateChangeSet` | `arn:aws:cloudformation:*:aws:transform/Serverless-2016-10-31` | SAM `Transform:` macro requires this |
| `lambda:*` lifecycle | `analyticshealth-ingestion-*` functions | SAM creates/updates functions |
| `lambda:GetLayerVersion` | `017000801446:layer:AWSLambdaPowertoolsPython*` | Powertools layer lives in AWS's account — CloudFormation validates it at changeset creation |
| `iam:*` | `analyticshealth-ingestion-*` roles | SAM creates execution roles per Lambda |
| `s3:PutObject/GetObject/ListBucket` | SAM managed bucket + CDK assets bucket | SAM uploads packaged code before deploy |
| `ssm:GetParameter` | `/analyticshealth/*` | SAM template reads SSM params at deploy time |
| `states:UpdateStateMachine/DescribeStateMachine` | `analyticshealth-*` | Step Functions ARN exported by IngestionStack |

### `analyticshealth-api-deploy` / `analyticshealth-ai-deploy`
Scoped to Lambda code updates only (`UpdateFunctionCode`, `PublishVersion`, `CreateAlias`, etc.) — no CloudFormation, no IAM.

---

## Workflow Files

### `analyticshealth-ingestion` — deploy (main)

```yaml
on:
  push:
    branches: [main]

jobs:
  test:
    steps:
      - run: pip install pytest 'moto[s3,dynamodb,secretsmanager]' pytest-cov pytz python-dateutil garminconnect
      - run: python -m pytest tests/ -v --cov=src --cov-report=term-missing

  deploy:
    needs: test
    environment: production        # manual approval gate
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_INGESTION_DEPLOY_ROLE_ARN }}
      - run: sam build
      - run: |
          sam deploy \
            --stack-name analyticshealth-ingestion-prod \
            --capabilities CAPABILITY_IAM \
            --resolve-s3 \
            --no-confirm-changeset \
            --no-fail-on-empty-changeset \
            --parameter-overrides Env=prod UserId=lucas-user-id ...
```

### `analyticshealth-ingestion` — diff (feature branches)

```yaml
on:
  push:
    branches: [feature/*]

jobs:
  diff:
    steps:
      - run: sam build
      - run: |
          sam deploy \
            --stack-name analyticshealth-ingestion-dev \
            --no-execute-changeset \          # preview only, no deploy
            --resolve-s3 \
            --no-fail-on-empty-changeset
```

---

## Secrets Required (GitHub Environments)

| Secret | Repo | Value |
|---|---|---|
| `AWS_INGESTION_DEPLOY_ROLE_ARN` | `analyticshealth-ingestion` | ARN exported by `analyticshealth-infra-oidc` |
| `AWS_INFRA_DEPLOY_ROLE_ARN` | `analyticshealth-infra` | ARN exported by `analyticshealth-infra-oidc` |
| `AWS_API_DEPLOY_ROLE_ARN` | `analyticshealth-api` | ARN exported by `analyticshealth-infra-oidc` |
| `AWS_AI_DEPLOY_ROLE_ARN` | `analyticshealth-ai` | ARN exported by `analyticshealth-infra-oidc` |

All ARNs are `CfnOutput` exports from the OIDC stack — retrieve them after the first `cdk deploy analyticshealth-infra-oidc`.

---

## Gotcha: `ROLLBACK_COMPLETE` State

A stack in `ROLLBACK_COMPLETE` cannot be updated — it must be deleted and recreated. This state occurs when a first-time deploy fails mid-execution and CloudFormation rolls back all changes.

```bash
aws cloudformation delete-stack \
  --stack-name analyticshealth-ingestion-prod \
  --region eu-west-1 --profile analyticshealth

aws cloudformation wait stack-delete-complete \
  --stack-name analyticshealth-ingestion-prod \
  --region eu-west-1 --profile analyticshealth
```

Then re-run the GitHub Actions workflow. See also [Gotchas](gotchas.md).
