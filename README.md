# n8n Automation Portfolio

Production-grade automation workflows by **Levelbrook Consulting** — built by a senior
software engineer, not a template-stitcher. Every workflow here is **real, importable
n8n JSON** you can drop into your own instance (`Import from File`) and inspect: the
architecture, the error handling, and the engineering decisions that keep it running
after handoff.

> The point of this repo is proof you can *run*, not screenshots. Import any workflow,
> wire up your own credentials, and see how it's built.

## The builds

| # | Workflow | What it does | Highlights |
|---|----------|--------------|-----------|
| **01** | [RAG Knowledge Agent](./01-rag-knowledge-agent) | Chat with your company's docs in Slack/Telegram; cited answers; auto re-index on doc change | Vector retrieval, citation-grounded agent, Whisper voice, **idempotent delete-then-insert re-indexing** |
| **02** | [Lead Enrichment & ICP Scoring](./02-lead-enrichment-scoring) | Inbound lead → enriched → LLM-scored → deduped → CRM → hot-lead alert | **Caching enrichment microservice**, structured-JSON LLM scoring, idempotent dedup, error observability |
| **03** | [Content Repurposing Engine](./03-content-repurposing) | One podcast/blog → LinkedIn + X thread + newsletter + video captions | Parallel channel generation, brand-voice prompting, **human-in-the-loop approval** |

## What makes these senior-grade

Most automation help stops at the node palette. These go further:

- **Custom code where n8n has no node** — small supporting microservices (enrichment
  cache, transcription) with real auth, batching, and rate-limit handling.
- **Reliability by design** — idempotency, dedup, retries, and failure alerting built in
  from the start, not bolted on after it breaks.
- **AI, grounded** — RAG with source citations and structured LLM extraction, wired into
  real systems instead of toy chatbots.

## Import instructions

1. In n8n: **Workflows → Import from File** → select the workflow's `workflow.json`.
2. Open each credential-referenced node and connect your own credentials
   (OpenAI/Anthropic, Supabase, Slack/Telegram, CRM, etc.). No secrets ship in this repo.
3. Read the per-workflow `README.md` for the architecture and any supporting services.

## Work with me

I build n8n & Make automations, AI-agent workflows, custom nodes, and Zapier/Make → n8n
migrations. Remote, async-friendly.

**→ Portfolio & case studies:** https://automation.levelbrook.com
**→ Get in touch:** levelbrookteam@gmail.com

*Levelbrook Consulting — automation engineering.*
