# AI Content-Repurposing Engine — *one input → ten outputs*

An n8n workflow that turns a single long-form recording (podcast, webinar, or
talk) into a full week of channel-ready content: a LinkedIn post, an X/Twitter
thread, a newsletter section, and three short-form video hooks — brand-checked,
human-approved, published, and logged.

> **Honesty note:** this is a self-built portfolio/demo workflow. There are no
> real clients, no usage numbers, and no invented metrics here — the sections
> below describe *what the workflow does*, not results it has produced.

---

## Problem

Teams record one great asset — an hour-long podcast or webinar — and then let it
die as a single link. Repurposing it by hand is the bottleneck: someone has to
listen back, pull quotes, rewrite for each platform's format and voice, get it
approved, publish it in four places, and track what went where. It rarely
happens consistently, so the best material never compounds.

The manual pipeline has four expensive steps that are all automatable:
transcription, distillation, per-channel rewriting, and the publish/log
bookkeeping. The one step that *shouldn't* be automated away is the final
sign-off — a human should still approve anything that goes out under the brand.

## Workflow

**Architecture (23 nodes):**

```
New Recording (Drive Trigger)
   └─ Download Media File (Google Drive)
        └─ Transcribe Audio  (OpenAI Whisper)
             └─ AI Agent: Extract Content Atoms  ──┐  structured JSON
                  ├─ OpenAI Chat Model  (ai_languageModel)
                  └─ Content Atom Schema (ai_outputParser, strict schema)
                        │
        ┌───────────────┴──── fan-out (parallel) ────────────────┐
   Generate       Generate       Generate           Generate
   LinkedIn Post  X Thread       Newsletter         Video Captions
        └──────────────┴──── Merge Channel Drafts ──┴────────────┘
                        └─ Brand-Safety Sanitizer (banned-keyword scan)
                             └─ IF Brand-Safe? ── no ─► Log Rejected/Held
                                   │ yes
                             Request Slack Approval
                                   └─ Wait for Approval (resume webhook)
                                        └─ IF Approved? ── no ─► Log Rejected/Held
                                              │ yes
                              ┌───────────────┼────────────────┐
                       Publish LinkedIn  Publish X Thread  Log to Content Ledger
```

### Key design decisions

- **Structured extraction before generation.** Instead of asking one prompt to
  "write everything," an **AI Agent** first distills the transcript into typed
  *content atoms* (title, summary, themes, verbatim quotes, stats, CTA). A
  **Structured Output Parser** sub-node enforces the JSON schema, so every
  downstream writer receives clean, predictable fields — and the atoms can't
  invent facts that weren't in the source. The chat model and the parser attach
  to the agent through n8n's LangChain `ai_languageModel` / `ai_outputParser`
  connections.

- **Parallel fan-out generation.** The agent's single output feeds **four
  channel writers at once** (one output → four targets), not a slow chain. Each
  writer is its own OpenAI node with a **channel-specific system prompt** plus
  shared **brand-voice guidelines** (tone, length, emoji/hashtag rules, "use
  only the supplied atoms"). Adding a fifth channel = one more node on the
  fan-out, no rewiring.

- **Brand-safety filter.** Before anything reaches a human, a Code node scans
  every variant for banned/off-brand keywords and sets a `brand_safe` flag. An
  IF node routes unsafe packs straight to the log as `held_brand_safety`,
  keeping obvious problems out of the reviewer's queue.

- **Human-in-the-loop approval.** Nothing auto-publishes blindly. Approved-by-
  filter drafts are posted to **Slack** with the source, title, and a resume
  URL; the **Wait** node pauses the execution until a reviewer approves (or
  rejects) via the resume webhook. Only an explicit `approved` decision unlocks
  the publish nodes.

- **Publish + log.** On approval, the LinkedIn and X nodes publish, and a
  **Google Sheets** "Content Ledger" append records source file, all four
  variants, status, flagged terms, and timestamp — one row per run, so nothing
  is lost and every asset is traceable. Rejected/held packs are logged too.

- **Credentials are references only.** Every credential is a placeholder
  reference (`REPLACE_WITH_CREDENTIAL_ID`) — no secrets are stored in the file.

## What it delivers

From one dropped recording, a single run produces:

- **1** LinkedIn post (hook → insight → soft CTA)
- **1** X/Twitter thread (5–8 numbered tweets)
- **1** newsletter section (headline + editorial blurb + "why it matters")
- **3** short-form video hooks/scripts with on-screen text overlays
- **1** Slack approval request and **1** ledger row logging every variant

…all drawn only from the source transcript, checked for banned keywords, and
gated behind a human sign-off before publishing. That's the "one input → ten
outputs" in practice: 6 finished creative assets plus the approval + logging
scaffolding around them.

---

## Import instructions

1. In n8n, open **Workflows → Import from File** (or paste via **… menu →
   Import from URL/Clipboard**) and select `workflow.json`.
2. Open each node that shows a credential warning and pick your own credential:
   - **Google Drive** (trigger + download): `googleDriveOAuth2Api`
   - **OpenAI** (Whisper, chat model, 4 writers): `openAiApi`
   - **Slack**: `slackOAuth2Api`
   - **LinkedIn**: `linkedInOAuth2Api`
   - **X / Twitter**: `twitterOAuth2Api`
   - **Google Sheets**: `googleSheetsOAuth2Api`
3. Replace the placeholder IDs:
   - `REPLACE_WITH_DRIVE_FOLDER_ID` — the Drive folder to watch
   - `REPLACE_WITH_SLACK_CHANNEL_ID` — the Slack channel for approvals
   - `REPLACE_WITH_SHEET_ID` — the Google Sheet used as the Content Ledger
4. (Optional) Tune the brand voice: edit the system prompts in the four
   `Generate …` nodes and the banned-keyword list in **Brand-Safety Sanitizer**.
5. Activate the workflow, then drop an audio/video file into the watched Drive
   folder to trigger a run. Approve the Slack message to publish.

### Notes / requirements

- Requires an n8n version with the LangChain nodes bundled (self-hosted n8n or
  n8n Cloud, mid-2024+). Node types used:
  `@n8n/n8n-nodes-langchain.agent`, `.lmChatOpenAi`, `.outputParserStructured`,
  `.openAi`.
- The Whisper transcription step uses OpenAI's `audio/transcribe`. For
  high-volume/lower-cost transcription you can swap it for an **HTTP Request**
  node pointed at a self-hosted Whisper microservice — the rest of the workflow
  is unchanged as long as the transcript text lands on the same field.
- The approval step uses a Slack message + **Wait (resume on webhook)**. If your
  n8n supports it, the Slack node's *Send and Wait for Response* operation can
  replace this pair for native approve/reject buttons.
