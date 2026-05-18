# RAG & Bedrock Layer

AnalyticsHealth uses a **Retrieval-Augmented Generation (RAG)** approach so users can ask free-form questions about their health history instead of navigating dashboards.

## Components

| Component | Choice |
|---|---|
| Knowledge Base | Amazon Bedrock Knowledge Base |
| Embedding model | Titan Embeddings v2 (1024 dimensions) |
| Vector store | pgvector on Amazon RDS PostgreSQL (`db.t4g.micro`) |
| Generation model | Claude 3.5 Sonnet via Bedrock (see [ADR-0003](../decisions/adr-0003-llm-claude-sonnet.md)) |
| Orchestration | Bedrock Agent (single agent, multi-tenant via session attributes) |

See [ADR-0001](../decisions/adr-0001-pgvector-vs-opensearch.md) for the vector store choice and [ADR-0005](../decisions/adr-0005-rds-provisioned-vs-aurora.md) for the RDS provisioned vs Aurora Serverless v2 follow-up.

## Knowledge Base Source Layout

```
s3://analyticshealth-data-{account}-{region}/knowledge-base/
└── {user_id}/
    └── {data_type}/
        └── YYYY.json     ← consolidated, LLM-optimised
```

Each document is the output of the **Consolidator Lambda** (Phase 4) which aggregates raw records into a denser, prose-friendly representation suitable for chunking and embedding.

## Consolidation Flow

```
S3 PUT raw/.../{user_id}/...
  → S3 event triggers Consolidator Lambda
      → aggregates raw data for that user
      → rewrites knowledge-base/{user_id}/{data_type}/YYYY.json
      → calls Bedrock KB StartIngestionJob (sync)
          → KB chunks, embeds, writes vectors to pgvector
              (each row tagged with metadata.user_id)
```

The ingestion job is asynchronous — the chat endpoint may briefly return slightly stale context after a new ingestion. Acceptable for the D-1 nature of the data.

## Retrieval Strategy

- **Single Knowledge Base** for all users
- **Metadata filter on `user_id`** applied on every `Retrieve` and `RetrieveAndGenerate` call
- Documents represent **consolidated time windows** (one year per data type per user) — small enough to chunk efficiently, large enough to capture trends

## Chat Flow

```
POST /chat
  Authorization: Bearer <Cognito JWT>
  Body: { "question": "How has my resting HR trended this month?" }

  → Lambda: Chat Handler
      → Cognito authorizer validates JWT, extracts user_id
      → DynamoDB.sessions: load last N messages for context
      → DynamoDB.users: load profile (age, weight, etc.)
      → Bedrock Agent.invokeAgent
          sessionAttributes: { user_id, name, age, current_weight }
          KB filter: { user_id: <jwt_user_id> }
          generation: Claude 3.5 Sonnet
      → DynamoDB.sessions: persist new exchange
  ← natural-language answer
```

`user_id` flows through three layers (JWT → sessionAttributes → KB metadata filter) so a single misconfiguration cannot leak another user's data.

## Why Not Dashboards
- LLMs explain trends, correlations and anomalies better than static charts
- Removes frontend complexity in MVP
- Allows ad-hoc questions ("is my HRV recovering after that flu?") that no fixed dashboard would surface

## Gotchas

- **KB sync is not instantaneous** — explicitly invoke `StartIngestionJob` from the Consolidator and accept short staleness on `/chat`
- **RDS cold start (~5–10 min) when stopped** — the first chat after the instance is stopped will fail until it boots. See [ADR-0005](../decisions/adr-0005-rds-provisioned-vs-aurora.md) for the stop/start schedule trade-off
- **Token budgeting** — Bedrock cost is the largest variable cost driver; alarm on monthly spend, not request count
