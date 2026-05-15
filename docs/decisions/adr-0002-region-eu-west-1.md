# ADR 0002 – AWS Region Selection

## Status
Accepted

## Context
The platform serves a user based in Portugal. The chosen region must offer:

- Amazon Bedrock with Claude 3.5 Sonnet
- Aurora Serverless v2 (PostgreSQL)
- Step Functions, Cognito, API Gateway
- Low latency from Portugal

## Options
1. **eu-west-1 (Ireland)** — closest European region with full service availability
2. **us-east-1 (N. Virginia)** — broadest service availability, but ~100ms additional latency
3. **eu-central-1 (Frankfurt)** — geographically close but farther than Ireland for Portugal

## Decision
Use **eu-west-1 (Ireland)** as the single deployment region.

## Rationale
- Lowest latency from Portugal (~15–20ms)
- Full availability of all required services including Bedrock with Claude 3.5 Sonnet
- GDPR-compliant data residency within the EU
- Competitive pricing (same as us-east-1 for most services)

## Consequences
- All infrastructure and data reside in eu-west-1
- Cross-region DR (if needed) would target eu-central-1
- No multi-region complexity in the MVP
