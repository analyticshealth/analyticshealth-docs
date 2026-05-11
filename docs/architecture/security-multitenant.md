# Multi-Tenant Security Model

Multi-tenancy is enforced using **defense in depth**.

## Isolation Layers
1. Cognito JWT → user identity
2. API-level validation of `user_id`
3. S3 prefix per user
4. DynamoDB partition keys per user
5. Knowledge Base metadata filtering

## Encryption
- Encryption at rest (AWS KMS)
- Encryption in transit (TLS 1.2+)

## Failure Model
A single misconfiguration should **never** expose another user's data.