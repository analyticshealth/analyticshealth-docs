# Multi-Tenant Security Model

Multi-tenancy is enforced using **defense in depth**: every layer is responsible for upholding `user_id` isolation independently. A misconfiguration in any single layer must never leak another user's data.

## Source of Truth for `user_id`

`user_id` is **always extracted from the validated Cognito JWT** inside the Lambda — never from the request body, query string, or headers.

```python
# Always do this
user_id = event["requestContext"]["authorizer"]["claims"]["sub"]

# Never do this
user_id = event["body"]["user_id"]  # ❌ user-controlled
```

API Gateway's Cognito authorizer guarantees the JWT signature and claims are valid before the Lambda runs.

## Isolation Layers

| Layer | Mechanism |
|---|---|
| Identity | Cognito User Pool, JWT signed by AWS |
| API | Cognito Authorizer on every route |
| Application | `user_id` extracted from JWT claims, propagated to every downstream call |
| S3 | `user_id` prefix on every object (`raw/{source}/{user_id}/...`, `knowledge-base/{user_id}/...`); IAM policies use `${aws:PrincipalTag/user_id}` conditions where applicable |
| DynamoDB | `user_id` is the partition key on `users`, `sessions`, `ingestion_control` — physically partitioned |
| Vector store | Every embedding row in pgvector has a `user_id` column; KB metadata filter enforces it on every `Retrieve` call |
| Bedrock Agent | `sessionAttributes.user_id` passed on every `invokeAgent`; the agent's instructions reference it |

## Secrets

- **Garmin OAuth tokens**: Secrets Manager, **one secret per user** (`analyticshealth/garmin/{user_id}`)
- **RDS master credentials**: Secrets Manager, rotated by AWS automatically
- **No secrets in environment variables or DynamoDB** — Secrets Manager is the only store

## Encryption

- **At rest**: AWS KMS Customer-Managed Key (`analyticshealth-storage`) for S3, DynamoDB, RDS, Secrets Manager; annual key rotation enabled
- **In transit**: TLS 1.2+ enforced (`enforceSSL: true` on the S3 bucket, HTTPS-only on API Gateway)

## Network

- RDS deployed in `PRIVATE_ISOLATED` subnets — no public endpoint, no NAT gateway
- VPC Gateway Endpoints for S3 and DynamoDB (free, traffic stays on AWS backbone)
- Lambdas that need RDS access run in-VPC; others (most ingestion Lambdas) run outside the VPC to avoid cold-start overhead

## Failure Model

A single misconfiguration should **never** expose another user's data. The non-negotiable invariants:

1. `user_id` originates only from a verified JWT
2. Every S3/DynamoDB/RDS access path includes a `user_id` predicate
3. Bedrock KB queries always carry the metadata filter
4. Cross-user IAM roles do not exist — there is no "service account" that can read all users

## Audit

- CloudTrail enabled (account default)
- CloudWatch structured JSON logs on every Lambda include `user_id` so per-user audit queries are trivial
