# AnalyticsHealth – Architecture Documentation

This repository contains the **technical architecture documentation** for **AnalyticsHealth**, a serverless, multi-tenant health analytics platform built on AWS.

The goal of this documentation is to:
- Serve as a **technical reference** for architecture and design decisions
- Demonstrate **cloud architecture thinking** aligned with the AWS Well-Architected Framework
- Act as a **portfolio artefact** for solution architecture roles

The documentation is published using **MkDocs + GitHub Pages**.

## Live Documentation
📘 https://analyticshealth.github.io/analyticshealth-docs/

## Key Characteristics
- Serverless-first (Lambda, Step Functions, S3, DynamoDB)
- Multi-tenant from day one
- Cost-optimised (<$30/month for early stage)
- RAG-based conversational analytics using Amazon Bedrock
- No dashboards — insights delivered in natural language

## Tech Stack (High Level)
- AWS Lambda, Step Functions, EventBridge, SQS
- Amazon S3 (data lake), DynamoDB (control plane)
- Amazon API Gateway, Amazon Cognito
- Amazon Bedrock (Claude 3.5 Sonnet + Knowledge Base)
- RDS PostgreSQL + pgvector (Phase 4)
- Amazon Textract (OCR pipeline)
- AWS CDK (TypeScript) + AWS SAM

> This repository intentionally contains **no application code**.