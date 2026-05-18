# Architecture Overview

AnalyticsHealth follows a **layered, event-driven architecture** optimised for low cost and operational simplicity.

## Deployment

- **Region**: `eu-west-1` (Ireland) — see [ADR-0002](../decisions/adr-0002-region-eu-west-1.md)
- **IaC**: AWS CDK (Python for Lambdas, TypeScript for infrastructure stacks)
- **CI/CD**: GitHub Actions with OIDC (no long-lived credentials)
- **Account model**: single AWS account, `dev` and `prod` separated by stack suffixes

## High-Level Flow

1. Data ingestion (Garmin export initial load, daily API sync, scale image OCR, manual input)
2. Raw JSON stored in S3 data lake partitioned by `user_id`
3. Consolidator aggregates raw data into Knowledge Base documents per user
4. Bedrock Knowledge Base syncs embeddings to pgvector on RDS PostgreSQL
5. Conversational `/chat` endpoint retrieves context and invokes Claude 3.5 Sonnet

## Core AWS Services

| Service | Role |
|---|---|
| AWS Lambda | Compute for all ingestion, consolidator and chat handlers |
| AWS Step Functions | Orchestration of daily Garmin ingestion with retry |
| Amazon EventBridge | Scheduled trigger (06:00 UTC daily) |
| Amazon S3 | Single data lake bucket with prefix-based zones |
| Amazon DynamoDB | Users, sessions (TTL 90d), ingestion idempotency control |
| Amazon Bedrock | Knowledge Base + Claude 3.5 Sonnet (see [ADR-0003](../decisions/adr-0003-llm-claude-sonnet.md)) |
| Amazon RDS PostgreSQL | pgvector vector store on `db.t4g.micro` — stoppable (see [ADR-0005](../decisions/adr-0005-rds-provisioned-vs-aurora.md)) |
| Amazon Cognito | User Pool — JWT-based auth for all API endpoints |
| Amazon API Gateway | REST API with Cognito authorizer |
| AWS KMS | CMK for S3, DynamoDB, RDS and Secrets encryption |
| AWS Secrets Manager | Garmin OAuth credentials per user |
| Amazon Textract | OCR backend for scale image extraction |

## CDK Stack Structure

```
analyticshealth-infra-oidc       ← GitHub OIDC roles (deploy first)
analyticshealth-infra-storage    ← S3, DynamoDB, RDS (pgvector), VPC, KMS
analyticshealth-infra-api        ← API Gateway, Cognito
analyticshealth-infra-ingestion  ← Step Functions, EventBridge (Lambdas via SAM)
analyticshealth-infra-ai         ← Bedrock KB, Agent (Phase 4)
```

## API Surface

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/chat` | Natural-language question against the user's health history |
| `POST` | `/ingest/garmin` | Manual trigger of Garmin sync |
| `POST` | `/ingest/weight` | Upload scale image for OCR |
| `POST` | `/ingest/manual` | Log a manual workout (e.g., jiu-jitsu) |
| `GET`  | `/history` | Retrieve session history |

All endpoints require a valid Cognito JWT; `user_id` is **always** extracted from the JWT, never from the request body.

## Design Tenets
- Stateless compute — all state in S3 / DynamoDB / RDS
- Explicit data ownership by `user_id` at every layer
- Idempotent ingestion — DynamoDB `ingestion_control` is the source of truth
- Pay-per-use everywhere; the only always-on resource (RDS) is stoppable during development
- Defense-in-depth for multi-tenant isolation — S3 prefix + DynamoDB partition + KB metadata filter + JWT-only `user_id`
