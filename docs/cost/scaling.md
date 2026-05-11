# Cost Scaling Considerations

## 10 Users
- Aurora still dominant fixed cost
- Linear increase in Bedrock usage

## 100 Users
- Consider:
  - Consolidation frequency tuning
  - Smaller context windows
  - Model fallback (Claude Haiku)

## Design Choice
Optimise for **idle cost**, not peak throughput.