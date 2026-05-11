# ADR 0001 – Vector Store Selection

## Status
Accepted

## Context
The platform requires vector similarity search for RAG.

## Options
1. OpenSearch Serverless
2. Aurora PostgreSQL + pgvector

## Decision
Use **Aurora Serverless v2 with pgvector**.

## Rationale
- ~10× cheaper at low scale
- Predictable idle cost
- Full control over schema
- Sufficient performance for <1k users

## Consequences
- Manual scaling considerations
- Requires SQL-level tuning