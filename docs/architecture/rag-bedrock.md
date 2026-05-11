# RAG & Bedrock Layer

AnalyticsHealth uses a **Retrieval-Augmented Generation (RAG)** approach.

## Components
- Amazon Bedrock Knowledge Base
- Titan Embeddings v2 (1024 dimensions)
- Aurora Serverless v2 + pgvector
- Claude 3.5 Sonnet for generation

## Retrieval Strategy
- Single Knowledge Base
- Metadata filter on `user_id`
- Documents represent consolidated time windows

## Why Not Dashboards?
- LLMs are better at explaining trends
- Reduces frontend complexity
- Allows flexible, ad-hoc questions