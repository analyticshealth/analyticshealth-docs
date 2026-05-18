# Roadmap

## ✅ Phase 0 — Infrastructure (Done)
- CDK stacks deployed: `storage`, `api`, `oidc`, `ingestion` (skeleton), `ai` (placeholder)
- Single S3 data lake bucket with KMS CMK, block public access, versioning
- DynamoDB: `users`, `sessions` (TTL 90d), `ingestion_control`
- RDS PostgreSQL `db.t4g.micro` for pgvector (stoppable, deletion protection) — see [ADR-0005](decisions/adr-0005-rds-provisioned-vs-aurora.md)
- VPC with Gateway Endpoints (S3, DynamoDB) — no NAT cost
- Cognito User Pool with strong password policy
- GitHub OIDC roles (no long-lived keys)
- ADRs: region (eu-west-1), LLM model (Claude 3.5 Sonnet), no fallback

## ✅ Phase 1 — Initial Load (Done)
- Local Python script to load full Garmin Connect export to S3
- Parsers for 6 data types: `activities`, `daily_summary`, `sleep`, `metrics`, `race_predictions`, `biometrics`
- Deduplication by `activityId` (activities) and `calendarDate` (all others)
- Hydration merged into `daily_summary`
- Idempotency via DynamoDB `ingestion_control` (batch check, 100 keys/call)
- `--dry-run` mode with per-type/per-year summary
- Shared `models.py` ready for Lambda reuse

## ✅ Phase 2 — Lambda Ingestion (Done)
- `fetch_garmin` Lambda — fetches all 6 data types for D-1 via Garmin Connect API
  - Credentials from Secrets Manager; retry with exponential backoff + 60s rate-limit sleep
  - Collects all per-type errors before raising — a `sleep` failure never blocks `metrics`/`biometrics`
- `ocr_weight` Lambda — Textract OCR pipeline for scale screenshots
  - Regex extractors with physiological range validation
  - Image always deleted in `finally` block; lifecycle rule as backstop
  - Date resolved in user timezone (not UTC) to avoid off-by-one at midnight
- `manual_ingest` Lambda — `POST /ingest/manual` with Cognito JWT validation
  - UUID suffix on S3 key + DynamoDB SK — multiple workouts per day are supported
- `template.yaml` SAM template with all 4 Lambdas (including consolidator placeholder)
- Shared modules: `s3_client.py`, `dynamo_client.py` with structured JSON logging, conditional writes

## 🔲 Phase 3 — Orchestration & Hardening (Next)
- Step Functions state machine (replace placeholder with real flow)
- Retry with exponential backoff + SQS DLQ
- Structured JSON logging + X-Ray tracing
- CloudWatch Alarms: Lambda errors, DLQ depth, Bedrock cost
- Secrets Manager for Garmin credentials

## 🔲 Phase 4 — Consolidator & RAG
- `consolidator` Lambda: aggregate raw → `knowledge-base/{user_id}/`
- Bedrock Knowledge Base with RDS pgvector (CDK AiStack)
- `/chat` endpoint with Claude 3.5 Sonnet + RAG retrieval
- Session history in DynamoDB
- Response streaming via Lambda Function URL

## 🔲 Phase 5 — Scale & Optimisation
- Multi-user onboarding flow
- Token budget controls per user
- S3 cross-region replication for DR (RPO 24h)
- Consolidation frequency tuning at 100+ users