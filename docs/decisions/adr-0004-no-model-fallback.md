# ADR 0004 – No LLM Model Fallback

## Status
Accepted

## Context
Some RAG architectures implement model fallback (e.g., Sonnet → Haiku) to reduce cost or handle throttling. This ADR evaluates whether a fallback is needed.

## Options
1. **Single model (Sonnet only)** — simpler architecture, consistent quality
2. **Fallback to Haiku** — lower cost on high-volume days, but adds routing logic
3. **Fallback to Titan** — cheapest, but significant quality drop in Portuguese

## Decision
**No model fallback.** Use Claude 3.5 Sonnet exclusively.

## Rationale
- At current scale (1–50 users, 2–20 queries/day), Sonnet cost is ~$2–5/month — fallback savings are negligible
- Fallback routing adds complexity: retry logic, quality degradation handling, user-facing inconsistency
- Haiku produces noticeably weaker answers for health trend analysis and correlation detection
- Bedrock throttling at this scale is extremely unlikely
- If cost becomes a concern at 100+ users, revisit with a new ADR — don't pre-optimise

## Consequences
- No model routing logic in the `/chat` Lambda
- Single `modelId` configuration parameter
- Simpler error handling (fail or retry same model)
- Must revisit if monthly Bedrock cost exceeds $50
