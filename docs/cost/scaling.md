# Cost Scaling Considerations

## 1 → 10 Users

- **RDS**: `db.t4g.micro` comfortably handles 10 light users; no change needed
- **Bedrock**: linear with chat volume — apply per-user monthly token budgets
- **Secrets Manager**: +$0.40/user for the Garmin secret
- Expected total: **~$30–50/month**

## 10 → 100 Users

- **RDS**: move to `db.t4g.small` or `db.t4g.medium` (~$15–30/mo) and stop stopping it — disruption is no longer acceptable
- **Consolidation frequency**: reduce per-user consolidator runs (batch nightly instead of per-PUT)
- **Bedrock**: enforce smaller context windows; cap responses with `maxTokens`
- **CloudWatch Logs**: tighten retention to 3 days, sample structured logs
- Expected total: **~$150–300/month** (dominated by Bedrock)

## 100 → 1000 Users

- **RDS**: shift to Aurora PostgreSQL (provisioned, multi-AZ); the always-on cost is now justified by load
- **Vector store**: evaluate moving from pgvector to OpenSearch Serverless — its higher floor (~$700/mo) becomes proportionally smaller per user
- **Consolidator**: rewrite as a streaming job, not Lambda-per-event
- **Bedrock**: consider Provisioned Throughput for predictable cost

## Design Choice

The architecture is optimised for **idle cost**, not peak throughput. This is correct for a long-tail of low-volume users and a small number of active ones. If usage patterns flip (steady high volume), reconsider Aurora Serverless v2 — its 0.5 ACU floor becomes negligible at steady-state load.
