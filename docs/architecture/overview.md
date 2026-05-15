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
3. Consolidator aggregates raw data into Knowledge Base documents
4. Bedrock Knowledge Base syncs embeddings to Aurora pgvector
5. Conversational `/chat` endpoint retrieves context and invokes Claude 3.5 Sonnet

## Core AWS Services

| Service | Role |
|---|---|
| AWS Lambda | Compute for all ingestion and chat handlers |
| AWS Step Functions | Orchestration of daily Garmin ingestion with retry |
| Amazon EventBridge | Scheduled trigger (06:00 UTC daily) |
| Amazon S3 | Single data lake bucket with prefix-based zones |
| Amazon DynamoDB | Users, sessions (TTL 90d), ingestion idempotency control |
| Amazon Bedrock | Knowledge Base + Claude 3.5 Sonnet (see [ADR-0003](../decisions/adr-0003-llm-claude-sonnet.md)) |
| Aurora Serverless v2 | PostgreSQL + pgvector for embedding storage |
| Amazon Cognito | User Pool — JWT-based auth for all API endpoints |
| Amazon API Gateway | REST API with Cognito authorizer |
| AWS KMS | CMK for S3, DynamoDB and Secrets encryption |
| AWS Secrets Manager | Garmin API credentials per user |

## CDK Stack Structure

```
analyticshealth-infra-oidc       ← GitHub OIDC roles (deploy first)
analyticshealth-infra-storage    ← S3, DynamoDB, RDS, VPC, KMS
analyticshealth-infra-api        ← API Gateway, Cognito
analyticshealth-infra-ingestion  ← Step Functions, EventBridge (Lambdas via SAM)
analyticshealth-infra-ai         ← Bedrock KB, embeddings (Phase 4)
```

## Design Tenets
- Stateless compute — all state in S3 / DynamoDB / Aurora
- Explicit data ownership by `user_id` at every layer
- Idempotent ingestion — DynamoDB `ingestion_control` as the source of truth
- Pay-per-use everywhere — no always-on servers except Aurora (min 0.5 ACU)
- Defense-in-depth for multi-tenant isolation — S3 prefix + DynamoDB partition + KB metadata filter