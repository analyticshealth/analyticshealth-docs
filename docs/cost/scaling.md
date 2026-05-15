# Cost Scaling Considerations

## 10 Users
- Aurora still dominant fixed cost
- Linear increase in Bedrock usage

## 100 Users
- Consider:
  - Consolidation frequency tuning
  - Smaller context windows
  - Token budget controls per user

## Design Choice
Optimise for **idle cost**, not peak throughput.