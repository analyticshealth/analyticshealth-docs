# ADR 0003 – LLM Model Selection

## Status
Accepted

## Context
The platform uses Amazon Bedrock for conversational RAG over health data in Portuguese. The model must:

- Understand health/fitness domain terminology
- Respond fluently in Portuguese
- Work well with RAG (grounded answers from retrieved context)
- Be cost-effective for personal use (2–20 queries/day)

## Options
1. **Claude 3.5 Sonnet** (`anthropic.claude-3-5-sonnet-20241022-v2:0`) — ~$3/M input, $15/M output
2. **Claude 3 Opus** — ~$15/M input, $75/M output
3. **Claude 3 Haiku** — ~$0.25/M input, $1.25/M output
4. **Amazon Titan Text** — lower cost but weaker multilingual performance

## Decision
Use **Claude 3.5 Sonnet** as the single generation model.

## Rationale
- Best cost/quality ratio for RAG conversational workloads
- Excellent comprehension of health data in Portuguese
- Available in eu-west-1 via Bedrock
- Estimated cost: ~$2–5/month for single-user usage
- Opus is 5× more expensive with marginal quality gain for this use case
- Haiku is cheaper but noticeably weaker at interpreting trends and correlations

## Consequences
- All `/chat` invocations use Sonnet — no model routing logic needed
- Token budget should be monitored via CloudWatch custom metrics
- Model version is pinned; upgrades require explicit ADR update
