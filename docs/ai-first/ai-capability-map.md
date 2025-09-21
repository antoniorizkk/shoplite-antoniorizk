# AI capability map — ShopLite (Week 2)

| Capability | Intent (user) | Inputs (this sprint) | Risk 1–5 (tag) | p95 ms | Est. cost/action | Fallback | Selected |
|---|---|---|---|---:|---:|---|:---:|
| Typeahead / Search suggestions | User wants product suggestions while typing a query to quickly reach product pages | Partial query, SKU titles, categories, synonyms (catalog only) | 2 | 300 | $0.0035 | Query-time lexical search / precomputed prefix index | ✅ |
| Support assistant (FAQ + Order status) | User asks about order status, returns, shipping, or policy questions | User message, Policies/FAQ markdown, order-status API by id | 3 | 1200 | $0.0475 | Escalate to human agent; canned templates | ✅ |
| Personalized homepage recommendations | User expects relevant product suggestions on homepage | User id (if available), basic session history, top-sellers | 3 | 500 | $0.02 (approx) | Generic top-sellers / category bestsellers | No |
| Visual search (image → product) | User uploads an image to find matching SKUs | User image (mobile), product images dataset | 4 | 1000 | $0.10+ (higher compute) | Text search by user-entered keywords | No |

## Why these two
Typeahead and the Support assistant directly impact two high-value KPIs: conversion (typeahead reduces time-to-product and improves click-through) and support contact rate / cost (assistant deflects repetitive tickets). Both reuse ShopLite's existing data sources (product catalog, Policies/FAQ, `order-status` API) which reduces integration risk and shortens prototype time. Latency and cost estimates are conservative and aligned with early production targets; both are feasible within the provided p95 goals using a mid-sized LLM (8B) plus targeted caching and retrieval.
