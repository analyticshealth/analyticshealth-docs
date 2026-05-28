# AnalyticsHealth

**AnalyticsHealth** is a serverless, multi-tenant health analytics platform that ingests objective and subjective health data and delivers insights through natural language using generative AI.

This documentation focuses on the **technical architecture**, trade-offs and design decisions behind the platform.

## What This Is
- A real-world, production-oriented architecture
- Cost-aware and designed for gradual scale
- Built following AWS Well-Architected principles

## What This Is Not
- A fitness app UI
- A real-time analytics platform
- A HIPAA-certified medical system

## Core Principles
- ✅ Serverless-first
- ✅ Multi-tenant by design
- ✅ Low idle cost
- ✅ Data quality > data volume
- ✅ Simplicity over premature optimisation

## Implementation Status

| Phase | Status | Description |
|---|---|---|
| 0 — Infrastructure | ✅ Done | CDK stacks: storage (S3 + DynamoDB + KMS + VPC), API (Cognito), OIDC, ingestion skeleton |
| 1 — Initial Load | ✅ Done | Local script to load full Garmin export history → S3 + DynamoDB idempotency |
| 2 — Lambda Ingestion | ✅ Done | `fetch_garmin`, `ocr_weight`, `manual_ingest` Lambdas + SAM template |
| 3 — Orchestration & Hardening | ✅ Done | Step Functions + EventBridge, SQS DLQ, CloudWatch Alarms, X-Ray, Powertools logging, GitHub Actions CI/CD |
| 4 — Consolidator + Bedrock | 🔲 Planned | RAG pipeline: consolidate → embed → chat (requires re-provisioning pgvector) |
| 5 — Scale & Optimisation | 🔲 Planned | Multi-user onboarding, token budget controls, S3 replication |