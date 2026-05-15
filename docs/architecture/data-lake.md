# Data Lake Design

Amazon S3 is used as a **logical data lake**, not a big data platform.

## Single Bucket, Prefix-Based Zones

All data resides in a single bucket: `analyticshealth-data-{account}-{region}`

```
s3://analyticshealth-data-{account}-{region}/
├── raw/
│   ├── garmin/{user_id}/YYYY/MM/DD/{data_type}.json
│   ├── weight/{user_id}/YYYY/MM/DD/biometrics.json   ← OCR output
│   └── manual/{user_id}/YYYY/MM/DD/{data_type}.json
├── processed/
│   └── {domain}/{user_id}/                           ← Phase 3
├── knowledge-base/
│   └── {user_id}/{data_type}/YYYY.json               ← Bedrock KB source
└── temp-uploads/
    └── {user_id}/weight/*.jpg                        ← TTL 1 day (deleted after OCR)
```

## Data Types (raw zone)

| Type | Description | Source |
|---|---|---|
| `daily_summary` | Steps, calories, HR, stress, hydration | Garmin UDSFile / API |
| `activities` | Per-activity detail — type, duration, distance, pace, HR | Garmin summarizedActivities / API |
| `sleep` | Duration, stages, score, SpO2, respiration | Garmin sleepData / API |
| `metrics` | VO2max, training load, status, fitness age | Garmin MetricsMaxMet + TrainingHistory |
| `race_predictions` | 5K/10K/half/marathon predicted times | Garmin RunRacePredictions |
| `biometrics` | Weight, BMI, body fat, muscle mass | Scale OCR / Garmin bioMetrics |

## Security

- **Encryption**: KMS CMK (`analyticshealth-storage`), key rotation enabled
- **Block public access**: BLOCK_ALL
- **SSL enforced**: `enforceSSL: true`
- **Versioning**: enabled on all zones
- **Tenant isolation**: S3 prefix per `user_id` — no cross-user prefix access granted
- **Temp uploads**: lifecycle rule expires `temp-uploads/` after 1 day; Lambda deletes explicitly after OCR

## Zones

- **Raw**: immutable, source-aligned, never overwritten — reprocessable from here
- **Processed**: cleaned, domain-oriented aggregations (Phase 3)
- **Knowledge Base**: LLM-optimised documents, chunked for Bedrock ingestion
- **Temp uploads**: scale images, short-lived, never replicated

## Rationale
- Single bucket simplifies IAM policies (prefix-based conditions)
- Extremely low cost — S3 Standard at ~200 MB/year for 100 users
- Native 11-nines durability
- Easy recovery and replay from raw zone
- Lifecycle rules handle cleanup automatically