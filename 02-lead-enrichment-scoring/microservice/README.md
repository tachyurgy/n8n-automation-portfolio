# `enrich-svc` — Company Enrichment Microservice

A small self-hosted HTTP service that sits between the n8n workflow and the paid
enrichment providers. It is the engineered component that makes the pipeline cheap,
resilient, and vendor-agnostic. n8n calls **one** stable endpoint; this service owns
caching, provider fan-out, normalization, and cost control.

> Spec for a real component. Framed as what it does — no external SLAs or usage
> numbers claimed.

## Why it exists (design rationale)

- **Cost control** — enrichment providers charge per lookup. Leads cluster by
  company, so the same domain is looked up repeatedly. Caching by domain collapses
  those to one paid call per TTL window; batching amortizes provider rate limits.
- **Anti-corruption layer** — Apollo, Clearbit, and Prospeo all return different
  shapes and different field names. This service merges them into **one** schema, so
  the workflow never sees a provider's quirks and a provider outage degrades
  gracefully instead of breaking the automation.
- **Vendor swap without workflow edits** — add/remove a provider here; the n8n
  contract is unchanged.

## Endpoint

### `GET /v1/company`

Enrich a company/contact by email domain.

**Auth:** `Authorization: Bearer <token>` (n8n HTTP Header Auth credential).

**Query params**

| Param    | Required | Description                                        |
|----------|----------|----------------------------------------------------|
| `domain` | yes      | Company email domain, e.g. `acme.com`.             |
| `email`  | no       | Full lead email; enables person-level enrichment.  |
| `fresh`  | no       | `true` bypasses cache and forces a provider fetch. |

**200 response (normalized schema)**

```json
{
  "domain": "acme.com",
  "company_name": "Acme Inc.",
  "industry": "B2B SaaS",
  "employee_count": 320,
  "estimated_revenue": "$25M-$50M",
  "tech_stack": ["Ruby on Rails", "Shopify", "Stripe", "AWS"],
  "country": "US",
  "person_title": "VP of Engineering",
  "linkedin_url": "https://www.linkedin.com/company/acme",
  "confidence": 0.86,
  "sources": ["apollo", "clearbit"],
  "cache": "hit",
  "enriched_at": "2026-07-11T18:22:04Z"
}
```

Unknown fields are returned as `null` rather than omitted, so the workflow's
expression references never throw. On a total miss the service returns `200` with the
firmographic fields `null` and `"sources": []` (the lead still flows through scoring
as a low-signal record) — it does **not** 4xx/5xx on "not found".

**Error responses**

| Status | Meaning                                        |
|--------|------------------------------------------------|
| `400`  | Missing/invalid `domain`.                      |
| `401`  | Missing/invalid bearer token.                  |
| `429`  | Caller rate limit exceeded (per-token bucket). |
| `502`  | All upstream providers failed (after retries). |

### `GET /healthz`
Liveness/readiness probe (`{ "ok": true }`), used by the container orchestrator.

## Internal behavior

1. **Normalize** the domain (strip `www.`, lowercase, drop freemail domains → returns
   an empty-firmographic record so gmail-only leads are handled explicitly).
2. **Cache lookup** — Redis key `enrich:company:<domain>`, TTL ~30 days
   (configurable). Hit → return immediately with `"cache": "hit"`.
3. **Provider fan-out** (miss) — query providers in a configured priority order
   (e.g. Apollo → Clearbit → Prospeo), with a short per-provider timeout and one
   retry; merge results field-by-field, preferring higher-confidence/higher-priority
   sources.
4. **Batching** — a background queue coalesces bursts of misses and respects each
   provider's rate limit; the sync endpoint returns as soon as the first sufficient
   result is available.
5. **Persist** the merged record to cache and return the normalized schema.

## Suggested stack

- Node/TypeScript (Fastify) or Python (FastAPI) — a single small container.
- Redis for the domain cache (and the batch/rate-limit buckets).
- Provider API keys held **only** in the service's environment — never in n8n. n8n
  holds just the bearer token to this service, shrinking the workflow's secret
  surface to one credential.
- Deploy behind the same private network as n8n (hence `enrich.internal`); expose
  only to the n8n host.

## Environment (example)

```
PORT=8080
AUTH_TOKENS=<comma-separated bearer tokens>
REDIS_URL=redis://redis:6379/0
CACHE_TTL_DAYS=30
APOLLO_API_KEY=...
CLEARBIT_API_KEY=...
PROSPEO_API_KEY=...
PROVIDER_ORDER=apollo,clearbit,prospeo
PROVIDER_TIMEOUT_MS=4000
```
