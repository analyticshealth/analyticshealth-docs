# Data Lake Design

Amazon S3 is used as a **logical data lake**, not a big data platform.

## Bucket Structure
s3://analyticshealth/
raw/{source}/{user_id}/YYYY/MM/DD/
processed/{domain}/{user_id}/
knowledge-base/{user_id}/consolidated/

## Zones
- **Raw**: immutable, source-aligned
- **Processed**: cleaned, domain-oriented
- **Knowledge Base**: LLM-optimised documents

## Rationale
- Extremely low cost
- Native durability
- Easy recovery and replay
- Simple lifecycle management