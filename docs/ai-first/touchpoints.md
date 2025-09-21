# Touchpoint specs — Selected AI touchpoints

## 1) Typeahead / Search suggestions

**Problem statement**  
Users abandon or take long paths when search requires full queries or exact matches. A responsive typeahead that returns relevant SKU suggestions while typing reduces time-to-product, increases click-through on product pages, and improves conversion.

**Happy path**  
1. User focuses the site search input and begins typing "wireless he...".  
2. Client sends prefix request to Typeahead service (includes session locale, partial query).  
3. Typeahead service checks cache keyed by prefix; if hit -> returns cached suggestions instantly.  
4. On cache miss, service runs prefix retrieval over catalog + fuzzy matching and calls the lightweight LLM prompt to rewrite/rerank top candidates.  
5. Service returns 5 suggestions (title + short meta + product link) to client.  
6. User taps a suggestion and lands on the product page; analytics logs suggestion impression and click.  
7. If the user continues typing, repeat step 2 with updated prefix; update cached set TTL.

**Grounding & guardrails**  
- Source of truth: product catalog (SKU titles, categories, availability) stored in the app database.  
- Retrieval scope: limit to top 500 candidate SKUs by lexical score, then use the model to rerank top 10.  
- Max context: only send the typed prefix + top 10 candidate metadata (≤ 256 tokens total).  
- Refuse outside scope: model must not invent products or prices. If the query references a non-existent SKU, return "No exact match — try related terms" and rely on lexical fallback.

**Human-in-the-loop**  
- Not required for day-to-day.  
- Exception: analytics-driven quality alerts — if suggestion CTR falls below threshold (configurable, e.g., CTR < 3% for 7 days for a given prefix set), create a ticket for a product/content reviewer. Reviewer SLA: triage within 48 hours.

**Latency budget (p95 ≤ 300 ms)**  
- Network (client → service) roundtrip: 60 ms  
- Cache lookup: 10 ms (cache hit path)  
- If cache miss: retrieval (lexical + top 500 filter): 50 ms  
- Model inference / rerank: 160 ms  
- Post-process + response: 20 ms  
- Total (cache hit): ~90 ms; (cache miss): ~300 ms (target p95)  
- Cache strategy: TTL cache by prefix (1 hour) + LRU eviction; precompute top suggestions for high-volume prefixes.

**Error & fallback behavior**  
- Cache error: fallback to server-side lexical search (Elasticsearch / DB prefix search).  
- Model error/timeouts: fall back to lexically ranked suggestions and return a "Best matches" header.  
- Log errors and increment an SRE alert if model timeouts exceed 1% of requests in 10 minutes.

**PII handling**  
- No PII should be included in suggestions. Strip any user-identifying tokens on the server before sending to the model.  
- Logging policy: store suggestion impressions and click events only with hashed session id; do NOT store raw queries tied to user identifiers for more than 30 days unless required for debugging (then redact to last 4 chars).

**Success metrics**  
- Product metric 1 (CTR on suggestions) = suggestion_clicks / suggestion_impressions.  
- Product metric 2 (Time-to-first-product-click) = median(time from search focus to product click) pre/post feature.  
- Business metric (conversion uplift) = (orders_from_suggestions / suggestion_clicks) - baseline_conversion_rate; or relative uplift = (conversion_with_suggestions - baseline) / baseline.

**Feasibility note**  
Catalog and SKU metadata are available (10k SKUs). Implementation requires a lightweight retrieval layer (prefix index / Elasticsearch) and a model endpoint (Llama 3.1 8B). Next prototype step: implement prefix cache + lexical suggestions, then add an LLM reranker for top 10 candidates and measure p95.

---

## 2) Support assistant (FAQ + order status)

**Problem statement**  
Customers contact support for order status, returns, shipping, and policy questions. A grounded assistant that answers FAQ intents and returns `order-status` by id can deflect repetitive tickets, shorten resolution time, and reduce cost-per-ticket, while escalating complex or sensitive queries to humans.

**Happy path**  
1. User opens support chat and asks: "Where is my order #123456?"  
2. Client sends request to assistant service with the user's message and optional order id entered by user.  
3. Assistant service retrieves the matching Policy/FAQ entries and calls `order-status` API to fetch the live state (if order id provided).  
4. System forms a grounded prompt including (a) redacted order-status summary, (b) top-3 relevant FAQ passages, (c) a short instruction template to answer clearly and cite the order id last 4 digits.  
5. LLM generates the reply. Assistant returns the answer including a short "what next" and a "Contact agent" CTA.  
6. System logs conversation (redacted) and analytics; if the assistant confidence is below threshold or the user asks to "speak to an agent", mark for escalation and create a ticket with transcript.  
7. If escalated, the agent receives a prefilled ticket with context and an SLA for first response.

**Grounding & guardrails**  
- Source of truth: Policies/FAQ markdown + `order-status` API (live). Only include order fields necessary to answer (status, estimated delivery, last4).  
- Retrieval scope: top 5 FAQ passages + order summary.  
- Max context: 6,000 tokens (prompt + retrieved docs + short conversation history); trim older history beyond that.  
- Refuse outside scope: the assistant declines to answer legal or medical claims and shows a standard refusal + agent contact option.

**Human-in-the-loop**  
- Escalation triggers:
  - Low model confidence / low retrieval score (e.g., confidence < 0.6).  
  - Presence of financial dispute keywords (refund > $500, chargeback).  
  - User explicit "speak to agent" request.  
- UI surface: "Transfer to agent" CTA in chat (includes full redacted transcript).  
- Reviewer/agent SLA: first agent triage within **1 hour** for escalations; full resolution per standard support SLA (configurable). Initial human confirmation of deviant replies is required before changing policies.

**Latency budget (p95 ≤ 1200 ms)**  
- Client network: 150 ms  
- Retrieval (FAQ + order-status call): 250 ms (order-status API included)  
- Model inference: 700 ms  
- Post-process / send: 100 ms  
- Total: ~1,200 ms (p95 target)  
- Cache strategy: cache FAQ retrieval results and order-status summaries for short TTLs (order status for 1–5 minutes, FAQs for 1 hour). Use response caching for identical clarifying questions when appropriate (30% cache hit assumed).

**Error & fallback behavior**  
- Order-status API failure: return a graceful message "We can’t fetch live status right now — would you like us to escalate?" and offer agent contact.  
- Model hallucination detected (low retrieval overlap): refuse and suggest agent transfer.  
- If the assistant fails 3 times in a session, auto-escalate.

**PII handling**  
- What leaves the app: only redacted order snippets (order id truncated to last 4 digits), status, and non-sensitive metadata. Never send full payment details, full addresses, or full identity numbers to the model.  
- Redaction rules: remove/replace credit card numbers, SSNs, and full addresses in messages before sending to the model. Preserve only minimal data necessary (e.g., last 4 digits).  
- Logging policy: store redacted transcripts and hashed order ids; maintain an audit log that notes redactions and who accessed full logs. Store logs encrypted at rest and rotate access credentials. Keep transcribed logs for at most 90 days unless flagged for investigation.

**Success metrics**  
- Product metric 1 (Deflection rate) = bot_handled_requests / total_support_requests.  
- Product metric 2 (Median time to first answer) = median(time from request to bot answer).  
- Business metric (Support cost savings) = deflection_rate * total_monthly_ticket_volume * avg_ticket_cost.

**Feasibility note**  
Policies/FAQ markdown and `order-status` API exist and are sufficient initial grounding sources. Implementation requires a retrieval layer (FAQ index), a small context window mapping, and a model endpoint (Llama 3.1 8B). Next prototype step: implement the "order status" happy path for authenticated users and measure deflection on a small sample (10% of traffic).
