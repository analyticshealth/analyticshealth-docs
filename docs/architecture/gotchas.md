# Project-Specific Gotchas

Subtle behaviours and traps that are easy to forget but expensive to rediscover.

## RDS Cold Start After Stop (~5–10 min)

The pgvector RDS instance is stoppable to save cost (~$2/mo stopped vs ~$14/mo running — see [ADR-0005](../decisions/adr-0005-rds-provisioned-vs-aurora.md)). When it is stopped:

- First `/chat` request after a long idle period will fail until the instance boots
- AWS auto-starts any stopped RDS instance after 7 days (cannot be disabled)
- Mitigation in production: schedule start/stop with EventBridge + Lambda based on usage patterns

## Bedrock Knowledge Base Sync is Not Instantaneous

After writing to `s3://.../knowledge-base/{user_id}/`, the embeddings are not searchable until `StartIngestionJob` completes (typically seconds to a minute).

- **Always invoke `StartIngestionJob` explicitly** from the Consolidator Lambda — do not rely on scheduled syncs
- Accept short staleness on `/chat` — the data is D-1 anyway

## Garmin Tokens: Secrets Manager Only

OAuth tokens are sensitive. **Never store them in DynamoDB**, even with KMS encryption — Secrets Manager provides rotation, auditing, and access policies that DynamoDB does not.

- One secret per user: `analyticshealth/garmin/{user_id}`
- The `fetch_garmin` Lambda reads its secret by name, derived from the `user_id` passed in by Step Functions

## OCR Output Must Be Range-Validated

Textract can hallucinate digits on small fonts or poor lighting. Every extracted field is validated against physiological ranges (e.g., weight 30–250 kg, body fat 2–60%) before persisting. Out-of-range values raise — they are never silently coerced.

## Ingestion Idempotency: `user_id` + `source#type#date`

The `ingestion_control` table is the **only** mechanism preventing duplicate ingestion:

- PK: `user_id`
- SK: `{source}#{data_type}#{YYYY-MM-DD}` (e.g., `garmin_api#sleep#2024-03-15`)
- Re-running any pipeline is safe — already-ingested records are skipped via conditional writes

## OCR Date in User Timezone

The scale image is timestamped in the user's local timezone (`Europe/Lisbon`), **not** UTC. A workout finished at 00:30 local would otherwise be filed under the wrong day. The `ocr_weight` Lambda resolves the date in `Europe/Lisbon` and stores it as `YYYY-MM-DD` in the record envelope.

## Scale Image Lifecycle: Two Defences

The scale image is sensitive (face may be visible in some scales). Two independent mechanisms ensure it never lingers:

1. The Lambda deletes the source object in a `finally` block (runs even on errors)
2. S3 lifecycle rule expires `temp-uploads/` after 1 day as a backstop

## Manual Workouts: Multiple Per Day

Two jiu-jitsu sessions on the same day must be stored independently. Both the S3 key and the DynamoDB sort key carry a UUID suffix — there is no "one workout per day" assumption.

## `user_id` is Never From the Request Body

The single most important security rule:

```python
# ✅
user_id = event["requestContext"]["authorizer"]["claims"]["sub"]

# ❌ user-controlled — would allow anyone with a valid JWT to query any user's data
user_id = event["body"]["user_id"]
```

Any code review that approves the latter form is rejected.
