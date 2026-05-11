# Architecture Overview

AnalyticsHealth follows a **layered, event-driven architecture** optimised for low cost and operational simplicity.

## High-Level Flow

1. Data ingestion (Garmin, images, manual input)
2. Raw data stored in S3 (data lake)
3. Consolidation and aggregation per user
4. Knowledge Base generation
5. Conversational queries via RAG

## Core AWS Services
- AWS Lambda
- AWS Step Functions
- Amazon S3
- Amazon DynamoDB
- Amazon Bedrock
- Amazon Aurora Serverless v2 (PostgreSQL + pgvector)
- Amazon Cognito
- Amazon API Gateway

## Design Tenets
- Stateless compute
- Explicit data ownership by `user_id`
- Idempotent ingestion
- Pay-per-use everywhere possible