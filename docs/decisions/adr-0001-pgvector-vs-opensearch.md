# ADR 0001 – Vector Store Selection

## Status
Accepted — superseded in part by [ADR-0005](adr-0005-rds-provisioned-vs-aurora.md) regarding the underlying RDS configuration. The pgvector decision still stands; only the host engine changed.

## Context
The platform requires vector similarity search for RAG. Bedrock Knowledge Base supports several backends: OpenSearch Serverless, Aurora pgvector, RDS pgvector, Pinecone, MongoDB Atlas, Redis Enterprise.

## Options
1. **OpenSearch Serverless** — AWS-native, purpose-built for search
2. **pgvector on PostgreSQL** (Aurora or RDS) — AWS-native, relational, low floor cost
3. **Pinecone / MongoDB Atlas / Redis** — external, free tiers available

## Decision
Use **pgvector on Amazon PostgreSQL**.

## Rationale

- **Idle cost dominates at MVP scale.** OpenSearch Serverless requires a minimum of 4 OCUs (2 indexing + 2 search) at ~$0.24/OCU-hour, giving a floor of **~$700/month** even with zero traffic. pgvector on a small PostgreSQL instance costs **<$20/month** for the same idle state.
- **Sufficient performance for <1k users.** pgvector with HNSW indexes handles tens of millions of embeddings on modest hardware. Our volume (~1.4MB/year/user of health data) produces far fewer than that.
- **Schema control.** Full SQL access lets us add custom columns (`user_id`, `data_type`, `period`) and filter at query time without relying on metadata-only filtering.
- **Stays in AWS.** No external accounts, API keys, or VPC peering. Bedrock KB integrates with RDS pgvector natively.

## Cost Note (Important)

The original version of this ADR stated pgvector via Aurora Serverless v2 would "pause when idle" with a cost of "~$2–4/month". **That was incorrect.** Aurora Serverless v2 does *not* scale to zero — only Aurora Serverless v1 (now deprecated) did. The realised cost on Aurora Serverless v2 (0.5 ACU minimum) is **~$52/month** even with no traffic. This discrepancy was caught after the first month of billing and resolved by switching to a stoppable RDS instance — see [ADR-0005](adr-0005-rds-provisioned-vs-aurora.md).

The pgvector-vs-OpenSearch trade-off, however, is unchanged: pgvector still wins on idle cost by an order of magnitude or more.

## Consequences

- Manual capacity planning required as user count grows (see [Cost Scaling](../cost/scaling.md))
- SQL-level tuning needed for HNSW index parameters at scale
- Database upgrades require maintenance windows (mitigated by RDS Blue/Green deployments)
- A second ADR ([ADR-0005](adr-0005-rds-provisioned-vs-aurora.md)) governs the choice between Aurora Serverless v2 and RDS Provisioned as the host engine
