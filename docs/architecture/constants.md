# Constants Pattern (`bin/constants.ts`)

All project-wide string values that would otherwise be hardcoded in CDK stacks are centralised in a single file: `analyticshealth-infra/bin/constants.ts`.

---

## Why

CDK stacks reference each other's names, IAM role ARNs, resource prefixes, and third-party account IDs. Without a single source of truth, these values scatter across five stack files and become inconsistent over time.

The specific trigger: the AWS Lambda Powertools layer (`017000801446:layer:AWSLambdaPowertoolsPython*`) is hosted in an AWS-owned account. If that account ID is hardcoded in `oidc_stack.ts` without a clear label, it reads as an unknown magic number. With a named constant it is self-documenting.

---

## The File

```typescript
// analyticshealth-infra/bin/constants.ts

export const PROJECT = 'analyticshealth';
export const DEFAULT_REGION = 'eu-west-1';

export const GITHUB = {
  org: PROJECT,
  repos: {
    infra:     `${PROJECT}-infra`,
    api:       `${PROJECT}-api`,
    ingestion: `${PROJECT}-ingestion`,
    ai:        `${PROJECT}-ai`,
  },
} as const;

/** Third-party AWS account IDs referenced in IAM policies. */
export const THIRD_PARTY_ACCOUNTS = {
  lambdaPowertools: '017000801446',
} as const;

export const DEFAULT_USER_ID = 'lucas-user-id';
```

---

## What Each Constant Controls

| Constant | Used in | What it drives |
|---|---|---|
| `PROJECT` | All 5 stack files | Every resource name, export name, and ARN prefix — e.g. `${PROJECT}-users` table, `${PROJECT}-infra-deploy` role |
| `DEFAULT_REGION` | `bin/analyticshealth.ts` | Fallback region if `CDK_DEFAULT_REGION` env var is not set |
| `GITHUB.org` | `lib/oidc_stack.ts` | OIDC trust policy `sub` claim: `repo:{org}/{repo}:*` |
| `GITHUB.repos.*` | `lib/oidc_stack.ts` | OIDC trust policy per role, role names, and CloudFormation export descriptions |
| `THIRD_PARTY_ACCOUNTS.lambdaPowertools` | `lib/oidc_stack.ts` | IAM policy for `lambda:GetLayerVersion` on the AWS Powertools layer |
| `DEFAULT_USER_ID` | `lib/ingestion_stack.ts` | EventBridge rule target input `{ user_id: DEFAULT_USER_ID }` for the daily Garmin sync |

---

## Usage Pattern

Stack files import only what they need:

```typescript
// lib/oidc_stack.ts
import { PROJECT, GITHUB, THIRD_PARTY_ACCOUNTS } from '../bin/constants';

const ingestionRole = new iam.Role(this, 'IngestionDeployRole', {
  roleName: `${PROJECT}-ingestion-deploy`,
  assumedBy: oidcPrincipal(GITHUB.repos.ingestion),
});

ingestionRole.addToPolicy(new iam.PolicyStatement({
  actions: ['lambda:GetLayerVersion'],
  resources: [
    `arn:aws:lambda:${this.region}:${THIRD_PARTY_ACCOUNTS.lambdaPowertools}:layer:AWSLambdaPowertoolsPython*`,
  ],
}));
```

```typescript
// lib/storage_stack.ts
import { PROJECT } from '../bin/constants';

this.usersTable = new dynamodb.Table(this, 'UsersTable', {
  tableName: `${PROJECT}-users`,
  // ...
});
```

---

## What Is NOT in constants.ts

The following are intentionally left as CDK tokens or environment variables, not constants:

| Value | Where it comes from | Why |
|---|---|---|
| AWS account ID | `this.account` (CDK pseudo-parameter) | Resolved at deploy time — differs between accounts |
| AWS region | `this.region` (CDK pseudo-parameter) | Resolved at deploy time — `DEFAULT_REGION` is only a local fallback |
| Lambda ARNs | CloudFormation `Fn::ImportValue` | Cross-stack exports, not known at synth time |
| Garmin secret ARN | SSM Parameter Store | Runtime value, not a build-time constant |

---

## Rule

> No hardcoded string that identifies this project, this account's third-party dependencies, or this organisation's GitHub repos may appear in any `lib/*.ts` or `bin/analyticshealth.ts` file. It must be sourced from `bin/constants.ts`.

The only exceptions are AWS service names and ARN components that are truly universal (e.g., `arn:aws:cloudformation:*:aws:transform/Serverless-2016-10-31`).
