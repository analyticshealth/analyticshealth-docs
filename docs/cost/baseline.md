# Cost Baseline

Target: **<$20/month** for single-user use during development.

## Monthly Estimates (Single User, eu-west-1)

| Service | Estimate | Notes |
|---|---|---|
| S3 | ~$0.01 | Total volume <50 MB/year |
| Lambda | ~$0.00 | Free tier covers single-user use |
| Step Functions | ~$0.01 | One execution per day |
| DynamoDB | ~$0.01 | Free tier; on-demand billing |
| **RDS PostgreSQL `db.t4g.micro`** | **~$2 stopped / ~$14 running** | Storage ~$2.30/mo (20GB gp3); compute stopped when not in use |
| Bedrock — embeddings (Titan v2) | ~$0.05 | Small consolidated docs |
| Bedrock — generation (Claude 3.5 Sonnet) | ~$2–5 | Depends on chat volume; largest variable cost |
| API Gateway | ~$0.01 | Free tier covers low traffic |
| Cognito | ~$0.00 | Free up to 50,000 MAU |
| KMS | ~$1.00 | 1× CMK ($1/month) |
| Secrets Manager | ~$0.40 per secret | RDS secret + 1 Garmin secret per user |
| CloudWatch Logs | ~$0.50 | With retention tuned to 7 days |
| **Total (dev, instance stopped most days)** | **~$6–10/month** | |
| **Total (always-on instance)** | **~$18–22/month** | |

## Historical Context

Previously the project ran on Aurora Serverless v2 (min 0.5 ACU). That floor cost ~$52/month *even when completely idle*, because Aurora Serverless v2 **does not scale to zero** (only Aurora Serverless v1, now deprecated, did). See [ADR-0005](../decisions/adr-0005-rds-provisioned-vs-aurora.md) for the migration to `db.t4g.micro`.

## What Drives Cost

1. **Bedrock token usage** — by far the largest variable cost as users actively chat
2. **RDS compute** — fixed ~$0.018/hour when running; $0 when stopped (storage only)
3. **KMS + Secrets Manager** — small, fixed, grows linearly with users

## What Does NOT Drive Cost (at this scale)

- S3 storage — health data is ~1.4MB/year per user
- DynamoDB capacity — on-demand, dominated by free tier
- Lambda — invocations and GB-seconds dominated by free tier
- API Gateway — request volume dominated by free tier

## Watch List

| Metric | Alarm Threshold |
|---|---|
| Monthly Bedrock spend | >$10/month/user |
| RDS uptime hours/month | >720h (i.e., never stopped) |
| Lambda errors | any sustained rate |
| DLQ depth | >0 |
