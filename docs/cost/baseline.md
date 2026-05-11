# Cost Baseline

Target: <$30/month for early-stage usage.

## Monthly Estimates (Single User)
- S3: ~$0.01
- Lambda: ~$0.00 (free tier)
- Step Functions: ~$0.01
- DynamoDB: ~$0.01
- Aurora Serverless v2 (pgvector): ~$5.00
- Bedrock (Claude): $2–5
- API Gateway: ~$0.01

## Key Cost Driver
- LLM usage (tokens)
- Vector store compute>