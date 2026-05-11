# Data Ingestion

The ingestion layer supports three data sources:

## Garmin (Automated)
- Daily scheduled ingestion via EventBridge
- Step Functions orchestrates retries and error handling
- Raw JSON stored in S3

## Body Composition (Image-based)
- Image uploaded via API
- Processed with Amazon Textract
- Validated and transformed into structured JSON
- Image deleted immediately after processing

## Manual Training Input
- JSON payload via API Gateway
- Stored as-is in raw zone

## Key Properties
- Idempotent ingestion using DynamoDB
- Per-user isolation
- Retry and DLQ for external API failures