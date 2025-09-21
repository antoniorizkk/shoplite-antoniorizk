# Cost model — Selected touchpoints

## Assumptions (base case)
- Model: **Llama 3.1 8B** via OpenRouter (chosen for cost/latency tradeoff).  
  - Prompt price = $0.05 / 1000 tokens.  
  - Completion price = $0.20 / 1000 tokens.
- Typeahead:
  - Avg tokens in: **10**  
  - Avg tokens out: **15**  
  - Requests/day: **50,000**  
  - Cache hit rate: **70%**
- Support assistant:
  - Avg tokens in: **150**  
  - Avg tokens out: **200**  
  - Requests/day: **1,000**  
  - Cache hit rate: **30%**

## Calculation (formula)
Cost/action = (tokens_in / 1000) * prompt_price + (tokens_out / 1000) * completion_price  
Daily cost = Cost/action * Requests/day * (1 - cache_hit_rate)

### Typeahead
- tokens_in = 10 → 10/1000 = 0.01 k tokens  
- tokens_out = 15 → 15/1000 = 0.015 k tokens  
- prompt cost = 0.01 * $0.05 = $0.0005  
- completion cost = 0.015 * $0.20 = $0.0030  
- **Cost/action = $0.0005 + $0.0030 = $0.0035**

Requests/day = 50,000; cache hit = 70% → model calls = 50,000 * 0.30 = 15,000  
- **Daily cost (Typeahead) = 15,000 * $0.0035 = $52.50**

### Support assistant
- tokens_in = 150 → 0.15 k tokens  
- tokens_out = 200 → 0.20 k tokens  
- prompt cost = 0.15 * $0.05 = $0.0075  
- completion cost = 0.20 * $0.20 = $0.0400  
- **Cost/action = $0.0075 + $0.0400 = $0.0475**

Requests/day = 1,000; cache hit = 30% → model calls = 1,000 * 0.70 = 700  
- **Daily cost (Support assistant) = 700 * $0.0475 = $33.25**

## Results
- **Typeahead**: Cost/action = **$0.0035** → Daily = **$52.50**  
- **Support assistant**: Cost/action = **$0.0475** → Daily = **$33.25**  
- **Combined daily cost** = **$85.75**  
- **Estimated 30-day cost** ≈ **$2,572.50**

> Annual rough figure = $85.75 * 365 ≈ **$31,298.75**

## Alternative (higher-quality model for critical paths)
If choosing **GPT-4o-mini** for support assistant (prompt $0.15/1K, completion $0.60/1K):
- Using same token estimates (150/200) and 700 model calls/day:
  - Cost/action ≈ $0.1425 → Daily ≈ **$99.75** → 30-day ≈ **$2,992.50**.
- Recommendation: reserve high-quality models for escalation or a small subset of complex queries to limit cost.

## Cost levers if over budget
- Increase cache hit rate (more precompute for high-volume prefixes; longer TTL for FAQs).  
- Shorten prompt context or aggressively trim conversation history.  
- Use a cheaper model (Llama 8B) for low-risk queries and route complex queries to a stronger model.  
- Precompute and serve top suggestions from a pre-built prefix index rather than calling the model for every miss.  
- Throttle or sample assistant traffic during peak hours for prototyping.

## Notes & next steps
- Measure real token counts from a small pilot (tokens in/out vary with localization and answer length). Replace estimates with measured averages before final budgeting.
