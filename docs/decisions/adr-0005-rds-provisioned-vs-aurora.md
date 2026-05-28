# ADR 0005 – RDS Provisioned vs Aurora Serverless v2

## Status
Accepted (2026-05-17). Supersedes the Aurora Serverless v2 implementation referenced in earlier versions of [ADR-0001](adr-0001-pgvector-vs-opensearch.md).

## Context

[ADR-0001](adr-0001-pgvector-vs-opensearch.md) selected pgvector on PostgreSQL as the vector store, but did not pin down whether the host should be **Aurora Serverless v2** or **RDS Provisioned**. The initial implementation used Aurora Serverless v2 with the assumption that it scales to zero when idle. After the first month of billing this assumption proved wrong:

- Aurora Serverless v2 has a **minimum capacity floor** of 0.5 ACUs that is billed 24/7
- Floor cost in `eu-west-1`: **0.5 ACU × $0.12/ACU-hour × 720h ≈ $43/month** for compute alone, plus storage and I/O — observed at **~$52/month** entirely idle
- Aurora Serverless v1 (deprecated) was the version that could pause to zero

The project is a personal/learning workload with usage measured in minutes per day. A $52/month idle floor was disproportionate.

## Options

1. **Stay on Aurora Serverless v2** — accept the ~$52/month floor as the cost of "managed"
2. **Switch to RDS Provisioned `db.t4g.micro` (stoppable)** — ~$14/month running, ~$2/month stopped (storage only)
3. **External vector store (Pinecone free tier)** — $0/month, but adds an external dependency outside AWS
4. **Destroy/restore Aurora on a schedule** — most ops overhead; complex for a learning project

## Decision

Use **RDS PostgreSQL `db.t4g.micro` (provisioned, stoppable)** as the pgvector host.

```typescript
new rds.DatabaseInstance(this, 'VectorDatabase', {
  engine: rds.DatabaseInstanceEngine.postgres({
    version: rds.PostgresEngineVersion.VER_15_15,
  }),
  instanceType: ec2.InstanceType.of(
    ec2.InstanceClass.BURSTABLE4_GRAVITON,
    ec2.InstanceSize.MICRO,
  ),
  // ...
});
```

## Rationale

- **Stoppable.** Unlike Aurora Serverless v2, a provisioned RDS instance can be stopped. Stopped state costs only storage (~$2/month for 20GB gp3). Development days without chat usage cost effectively nothing.
- **Stays in AWS.** No external accounts (rules out Pinecone for now), no VPC peering, native Bedrock KB integration.
- **Cost-appropriate to scale.** At single-user volumes the throughput of `t4g.micro` (2 vCPU burst, 1GB RAM) is vastly more than required.
- **Easy upgrade path.** When usage justifies it, switch to `t4g.small` / `t4g.medium` with a one-line CDK change and a maintenance window.

## Consequences

### Operational

- **Manual stop/start required during development.** AWS auto-starts any stopped RDS instance after **7 days** (cannot be disabled). Either accept a weekly restart or schedule a Lambda to stop the instance every Monday morning.
- **Cold start ~5–10 minutes** when restarting from stopped state. First `/chat` after a stop will fail until ready. Not acceptable in production — but acceptable while the project has zero real users.
- **No native auto-scaling.** Capacity decisions are explicit, not automatic.

### Lost vs Aurora Serverless v2

- No fast (~seconds) auto-scaling on demand spikes
- No automatic failover at the database layer (single-AZ on `t4g.micro`)
- No Aurora Backtrack / Global Database features

These trade-offs are acceptable for the MVP/learning phase. The [Cost Scaling](../cost/scaling.md) doc captures the trigger points to revisit this decision.

## Migration

A one-time migration is required for any existing Aurora cluster:

1. Snapshot the Aurora cluster
2. Export to S3 (or `pg_dump` from a temporary read replica)
3. Restore to the new RDS instance
4. Re-run Bedrock KB ingestion (rebuilds embeddings)
5. Destroy the Aurora cluster (deletion protection must be disabled first)

For the current state of the project (Phase 4 not yet implemented, no embeddings stored), the migration is a clean tear-down and re-create — no data to preserve in the vector store.

## Revisit Triggers

- Sustained chat traffic >100 requests/day (cold-start cost outweighs idle savings)
- Multi-user onboarding crossing 10 active users (single-AZ no longer acceptable)
- Cost of `t4g.medium` always-on approaches Aurora Serverless v2 steady-state cost

---

## Addendum — Instance Destroyed (2026-05-28)

The RDS `db.t4g.micro` instance provisioned under this ADR was **manually destroyed** during Phase 2/3 development to eliminate the ~$2/month storage cost during a period with no chat usage and no pgvector embeddings stored.

The CDK `StorageStack` no longer provisions any RDS resource. The `AiStack` retains the VPC reference for when Phase 4 requires a database.

### State as of 2026-05-28

| Component | State |
|---|---|
| RDS instance | Destroyed |
| VPC (PRIVATE_ISOLATED subnets) | Deployed — ready for reuse |
| `StorageStack` RDS block | Removed from CDK code |
| pgvector embeddings | None stored (Phase 4 not started) |

### Re-provisioning for Phase 4

When Phase 4 (Bedrock KB) begins, a new RDS instance must be provisioned. The options remain as described in this ADR. The cheapest and simplest path is to re-add the `DatabaseInstance` block to `StorageStack` and pass it to `AiStack`.

The decision of whether to use RDS Provisioned again or revisit Aurora Serverless v2 / an alternative vector store should be re-evaluated at that time. Key inputs:
- Aurora Serverless v2 minimum capacity pricing may have changed
- Bedrock Knowledge Base native vector store options (if any) should be checked
- Pinecone or Weaviate free tiers may now be viable if AWS-only is not a requirement
