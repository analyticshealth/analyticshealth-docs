# Data Ingestion

The ingestion layer supports four ingestion paths: initial load (one-time), daily Garmin sync, scale image OCR, and manual entry.

## Initial Load (one-time, local script)

Loads the full Garmin Connect export (~2–4 years of history) directly from a local unzipped export folder to S3.

```bash
# Dry run — validate without uploading
python -m initial_load.run_initial_load \
  --export-dir /path/to/garmin/export \
  --dry-run

# Real run
python -m initial_load.run_initial_load \
  --export-dir /path/to/garmin/export \
  --bucket analyticshealth-data-{account}-eu-west-1
```

**Parsed sources**:

| Source folder | Files | Output type |
|---|---|---|
| `DI-Connect-Fitness/summarizedActivities/` | 23 part files | `activities` |
| `DI-Connect-Aggregator/UDSFile_*.json` | 16 files | `daily_summary` |
| `DI-Connect-Aggregator/HydrationLogFile_*.json` | 16 files | merged into `daily_summary.hydration_ml` |
| `DI-Connect-Wellness/*_sleepData.json` | 16 files | `sleep` |
| `DI-Connect-Metrics/MetricsMaxMetData_*.json` | 16 files | `metrics` |
| `DI-Connect-Metrics/TrainingHistory_*.json` | 16 files | merged into `metrics` |
| `DI-Connect-Metrics/RunRacePredictions_*.json` | 16 files | `race_predictions` |
| `DI-Connect-Wellness/*_userBioMetrics.json` | variable | `biometrics` |

**Deduplication**: activities by `activityId`; all other types by `calendarDate`.

**Idempotency**: every uploaded record is registered in `analyticshealth-ingestion-control` with key `garmin_initial_load#{data_type}#{YYYY-MM-DD}`. Re-running skips already-ingested dates.

## Garmin Daily Sync (automated Lambda)

- **Trigger**: EventBridge Schedule → Step Functions → `fetch_garmin` Lambda (06:00 UTC)
- **Data**: D-1 for all 6 data types via Garmin Connect API
- **Auth**: credentials from Secrets Manager (per-user)
- **Idempotency**: same `ingestion_control` table, source key `garmin_api#{data_type}#{date}`
- **Resilience**: Step Functions retry with exponential backoff; failed executions sent to SQS DLQ *(Phase 3)*

## Body Composition OCR (image-based)

- **Trigger**: S3 PUT on `temp-uploads/{user_id}/weight/*.jpg`
- **Pipeline**: S3 event → `ocr_weight` Lambda → Textract `DetectDocumentText` → parse → validate ranges → write `biometrics` to `raw/weight/{user_id}/` → **delete source image**
- **Image retention**: never persists beyond processing. Lifecycle rule expires `temp-uploads/` after 1 day as safety net.

## Manual Training Input

- **Endpoint**: `POST /ingest/{data_type}` with Cognito JWT
- **Supported types**: any valid `data_type` (e.g. `activities` for jiu-jitsu, mobility)
- **Validation**: schema checked in Lambda before writing to `raw/manual/{user_id}/`

## Canonical Record Schema

All records written to S3 share the same envelope:

```json
{
  "user_id": "lucas-user-id",
  "date": "2024-03-15",
  "source": "garmin_initial_load | garmin_api | manual",
  "data_type": "daily_summary | activities | sleep | metrics | race_predictions | biometrics",
  "ingested_at": "2026-05-15T10:00:00Z",
  "payload": { ... }
}
```

Defined in `src/shared/models.py` (`GarminRecord` + per-type payload dataclasses).

## Idempotency Model

DynamoDB table `analyticshealth-ingestion-control`:

| Key | Value |
|---|---|
| PK `user_id` | `lucas-user-id` |
| SK `source_date` | `garmin_initial_load#daily_summary#2024-03-15` |
| `s3_path` | `raw/garmin/lucas-user-id/2024/03/15/daily_summary.json` |
| `status` | `completed` |
| `record_count` | `1` |