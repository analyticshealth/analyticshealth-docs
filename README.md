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
- AWS Lambda, Step Functions, API Gateway
- Amazon S3 (data lake)
- DynamoDB (control plane)
- Amazon Bedrock (Claude 3.5 Sonnet)
- Aurora Serverless v2 (PostgreSQL + pgvector)
- Amazon Cognito
- AWS CDK (Python)

> This repository intentionally contains **no application code**.