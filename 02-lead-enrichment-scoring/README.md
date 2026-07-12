# Inbound Lead-Gen, Enrichment & ICP-Scoring Pipeline (n8n)

A self-built, importable n8n workflow. Every inbound lead is normalized, deduped
against the CRM, enriched with firmographics, **LLM-scored against an Ideal Customer
Profile**, upserted to the CRM with the score attached, and — when it's a hot lead —
posted to a sales channel within seconds. A separate error-trigger flow gives the
whole pipeline observability.

> This is a portfolio / demonstration workflow. It ships with credential
> **references only** (no secrets) and describes what the automation *does*; it is
> not tied to any named client and contains no invented performance figures.

---

## Problem

Inbound leads arrive from a web form, Typeform, or a "book a demo" widget and then
sit. The realistic failure modes:

- **Duplicates** — the same person fills the form twice and you get two CRM contacts,
  splitting their history and triggering double outreach.
- **No context at the moment of arrival** — a raw `name + email` tells sales nothing
  about company size, industry, or stack, so triage is manual and slow.
- **No prioritization** — every lead looks the same in the inbox, so the genuinely
  high-fit ones don't get a fast human response (the single biggest driver of
  conversion for inbound).
- **Enrichment cost & lock-in** — hitting Apollo/Clearbit/Prospeo directly from the
  automation burns credits on repeat domains and hard-wires you to one vendor's API
  shape.
- **Silent failures** — an enrichment 500 or a CRM auth expiry drops leads on the
  floor with nobody noticing.

## Workflow (architecture & key decisions)

```
Inbound Webhook
   -> Normalize Lead Fields            (lowercase email, derive domain, name, company)
   -> CRM Dedup Search (by email)      (idempotency lookup BEFORE any write)
   -> IF Lead Already in CRM?
         true  -> Mark: Update Existing (carry crmContactId, mode=update)
         false -> Mark: New Contact     (mode=create)
   -> Enrich Company (self-hosted microservice)   <-- the differentiator
   -> Score Lead Against ICP           (LangChain Agent)
         + OpenAI Chat Model (GPT)         via ai_languageModel
         + ICP Score Parser (structured)   via ai_outputParser  -> {score,tier,reasoning}
   -> Upsert Contact to CRM (HubSpot)  (maps score/tier/reasoning onto the contact)
   -> Route by Tier (Switch)
         hot  -> Slack: Hot Lead Alert (name, company, score, reasoning, CRM link)
         warm -> Queue for Nurture
         cold -> Archive (no alert)
   -> Respond to Webhook (200)

On Workflow Error (Error Trigger)
   -> Format Error Payload -> Log Failure (ops store) -> Slack: Pipeline Error Alert
```

### 1. Idempotent dedup (no duplicate contacts, safe to replay)
The pipeline **searches the CRM by normalized email before it writes anything**. An
existing match routes to an update path carrying the existing `crmContactId`; a miss
routes to a create path. Because the final step is an `upsert` keyed on email, a
double form submission or a retried execution converges on **one** contact instead of
creating duplicates — the workflow is safe to re-run.

### 2. Enrichment via a self-hosted microservice (the senior decision)
The workflow does **not** call Apollo/Clearbit/Prospeo directly. It calls one internal
endpoint — `GET https://enrich.internal/v1/company?domain=` — that acts as an
anti-corruption layer:

- **Caches by domain** (Redis, TTL'd) so repeat leads from the same company cost
  nothing and return instantly.
- **Fans out** to multiple providers and merges/normalizes their responses into one
  stable schema, so a provider outage or a field rename never reaches n8n.
- **Batches / rate-limits** provider calls to control credit spend.
- Gives n8n a **single, stable contract** — you can swap or add enrichment vendors
  with zero workflow edits.

Full API contract in [`microservice/README.md`](./microservice/README.md).

### 3. Structured LLM scoring (typed output, not free text)
Scoring uses a LangChain **Agent** node wired to an OpenAI chat model *and* a
**Structured Output Parser**. The parser enforces a JSON schema
(`{score:int 0-100, tier:"hot"|"warm"|"cold", reasoning:string}`), so downstream nodes
consume `{{ $json.output.score }}` / `.tier` / `.reasoning` deterministically — no
regex-scraping of prose, no "the model returned markdown" breakage. The ICP itself
(target firmographics + disqualifiers) lives in the prompt and is the one place to
tune targeting.

### 4. Error observability
A second trigger — n8n's **Error Trigger** — catches a failure in *any* node,
formats a structured payload (workflow, failing node, message, execution URL), writes
it to an ops log store, and alerts `#ops` on Slack. Set this workflow as the
**Error Workflow** in n8n settings so a dropped lead is never silent.

## What it delivers

- Inbound leads are deduped, enriched, scored, and in the CRM **without manual triage**.
- Sales gets a rich Slack ping on hot leads **in seconds**, with the reasoning and a
  direct CRM link — so the fastest human follow-up goes to the best-fit leads.
- Enrichment cost is controlled by domain-level caching and batching in one owned
  component you can evolve independently of the workflow.
- Every scored lead carries a numeric `icp_score` + tier on its CRM record, so lists,
  routing, and reporting can all key off it.
- Failures are logged and alerted instead of vanishing.

---

## Import instructions

1. In n8n: **Workflows → ⋯ (top-right) → Import from File** and select
   `workflow.json`. (Or **Import from URL** if you host it.)
2. Open each node that shows a credential and pick/create the matching credential.
   The file ships with **references only** — no secrets:
   - **HubSpot** (`Upsert Contact to CRM`, `CRM Dedup Search`) — HubSpot App Token.
   - **OpenAI** (`OpenAI Chat Model (GPT)`) — OpenAI API key.
   - **Slack** (`Hot Lead Alert`, `Pipeline Error Alert`) — Slack OAuth2; set real
     channel IDs (placeholders `C0SALES` / `C0OPS`).
   - **Enrichment service** (`Enrich Company…`) — HTTP Header Auth for
     `enrich.internal` (see microservice spec).
3. Stand up (or stub) the enrichment microservice at the base URL used by the
   `Enrich Company` node, and create the custom CRM properties `icp_score`,
   `icp_tier`, `icp_reasoning`, `lead_source`.
4. Activate the workflow, copy the **production Webhook URL** from `Inbound Lead
   Webhook`, and point your form/Typeform at it (POST JSON:
   `{ "name", "email", "company", "source" }`).
5. In **Settings → Error Workflow**, select this workflow so the Error Trigger fires
   on failures.

### Swapping CRMs
The dedup + upsert are isolated to the HubSpot/HTTP nodes. To use Airtable or
Pipedrive, replace those three nodes and keep the normalize → enrich → score → route
spine unchanged.

## Node inventory (23 nodes)

Verified node types used: `webhook`, `set` (Edit Fields), `httpRequest`, `if`,
`switch`, `hubspot`, `slack`, `noOp`, `respondToWebhook`, `errorTrigger`,
`stickyNote`, plus LangChain `agent`, `lmChatOpenAi`, and `outputParserStructured`.
JSON validated with `python3 -m json.tool` / `json.load`.
