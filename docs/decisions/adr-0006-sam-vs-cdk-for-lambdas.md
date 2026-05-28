# ADR 0006 – SAM for Lambda Deployment vs CDK-only

## Status
Accepted (2026-05-28).

## Context

The project uses **AWS CDK (TypeScript)** for infrastructure stacks (S3, DynamoDB, VPC, API Gateway, Cognito, Step Functions, IAM). A decision was required on how to deploy the **Lambda functions** themselves — specifically the four ingestion Lambdas (`fetch_garmin`, `ocr_weight`, `manual_ingest`, `consolidator`).

Two realistic options existed:

1. **AWS SAM** — a separate template (`template.yaml`) processed by the SAM CLI, with its own deploy lifecycle (`sam build`, `sam deploy`)
2. **AWS CDK only** — define Lambda functions directly inside CDK stacks, co-located with the rest of the infrastructure

## Options

### Option 1 — SAM

- Lambdas defined in `template.yaml` with `Transform: AWS::Serverless-2016-10-31`
- Deployed via `sam build && sam deploy` — separate from CDK
- Lives in a separate repository (`analyticshealth-ingestion`)
- Exports Lambda ARNs via CloudFormation outputs, imported by the CDK `IngestionStack`
- CI/CD: dedicated GitHub Actions workflow per repo (test → diff on feature, test → deploy on main)

### Option 2 — CDK only

- Lambda functions defined in `IngestionStack` alongside Step Functions and EventBridge
- Single `cdk deploy` covers everything
- All infra in one repository

## Decision

Use **SAM for Lambda deployment** (Option 1).

## Rationale

**Separation of concerns between infra and application code.** The Lambda source code (`src/`) changes at a different cadence and has different reviewers than the infrastructure stacks. Keeping them in separate repos with separate CI/CD pipelines means a Python bugfix in `fetch_garmin` does not require a `cdk deploy` of the infrastructure stacks and vice versa.

**Independent deploy velocity.** The SAM workflow (test → build → deploy) takes ~90 seconds. A full `cdk deploy --all` takes ~3 minutes and touches resources that should rarely change. Developers pushing Lambda code changes should not wait for a CDK synthesis and changeset on stacks they did not touch.

**Role isolation.** Each repo has its own IAM deploy role with least-privilege permissions. The `analyticshealth-ingestion-deploy` role can create/update Lambda functions and SAM-managed CloudFormation stacks but has no access to CDK bootstrap resources or other stacks. A `cdk deploy` from the ingestion repo would require broader permissions, blurring the security boundary.

**SAM `sam local invoke` for development.** SAM provides `sam local invoke` and `sam local start-api` for local Lambda testing without deploying. CDK does not have an equivalent for Lambda execution.

## Consequences

### Cross-stack dependency

The CDK `IngestionStack` imports Lambda ARNs via `Fn::ImportValue`:

```typescript
const fetchGarminFn = lambda.Function.fromFunctionArn(
  this, 'FetchGarminFn',
  cdk.Fn.importValue(`${PROJECT}-fetch-garmin-fn-arn-${props.deployEnv}`),
);
```

**Deploy order is mandatory**: SAM must deploy before `cdk deploy analyticshealth-infra-ingestion`. If the CDK stack is deployed before the SAM stack, CloudFormation will fail with `Export not found`.

### Two CI/CD pipelines to maintain

Each change that spans Lambda code and infrastructure (e.g., adding a new Lambda and wiring it to Step Functions) requires coordinated PRs in two repositories and a specific deploy order (SAM first, then CDK infra).

### `ROLLBACK_COMPLETE` risk on first deploy

A failed first-time SAM deploy leaves the stack in `ROLLBACK_COMPLETE`. The stack must be deleted before retrying. This is a standard CloudFormation behaviour, not specific to SAM — but it surfaces more often on first deploys where IAM permissions are still being tuned.

## Alternatives Considered

**CDK `NodejsFunction` / `PythonFunction` constructs** — CDK's Lambda constructs can bundle and deploy function code, removing the two-repo split. Rejected because: (a) the Python bundling in CDK requires Docker during `cdk synth`, adding a dev dependency; (b) it collapses the infra/app boundary that the current repo structure enforces.

**Terraform + Lambda ZIP** — would unify the tool chain but introduce a third IaC language (HCL) into a project already committed to CDK. Rejected.
