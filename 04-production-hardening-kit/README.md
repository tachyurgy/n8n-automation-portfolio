# n8n Production Hardening Kit

Four importable workflows that close the gap between "every node is green" and "the work got
done". No SaaS, no external monitor, nothing to sign up for: three small workflows on your own
instance plus one worked example of how to wire them in.

They exist because the same failure keeps coming up on the n8n community forum: a token expired,
a field got renamed, a filter stopped matching, a tool inside an agent failed and the model
guessed. Every execution succeeded. Zero records landed. Nobody found out for three days, because
the Error Workflow only fires when something *throws*, and nothing threw.

| file | what it is | attach it where |
|---|---|---|
| `error-router.json` | Error Trigger → normalize → **dedup by workflow+node+message for 30 min** → Slack with a link to the execution | Workflow settings → Error Workflow, on every production workflow |
| `assert-outcome-subworkflow.json` | Sub-workflow: `actual` vs `min`/`max` → returns a receipt, or **Stop and Error** so the caller's execution fails loudly | Called with Execute Workflow right after any write |
| `execution-auditor.json` | Hourly: pulls recent executions through the n8n API node and flags **error streaks**, **all-failed windows** and **fast-green** workflows (successes that finish in milliseconds while the runs that did real work failed) | Runs on its own; needs an n8n API credential |
| `example-hardened-pipeline.json` | The fetch-and-append every tutorial shows, with the kit applied: retry + timeout + error output on HTTP, zero-rows guard, outcome assertion, error workflow set | Read it, copy the shape |

Verified: all four import cleanly into n8n **2.39.8** (`n8n import:workflow`), and each scores
100/100 on [n8n-workflow-lint](https://github.com/tachyurgy/n8n-workflow-lint).

## Import

n8n → Workflows → **Import from File** → pick each JSON. Then:

1. **Error Router** — choose your Slack credential and channel in the Slack node. Open every
   production workflow → Settings → *Error Workflow* → Error Router. (The Auditor and the example
   already reference it by a placeholder id; re-pick it in their settings after import.)
2. **Assert Outcome** — nothing to configure. In your own workflows add an *Execute Workflow*
   node after the write step, pick this workflow, and map `label`, `actual`, `min`, `max`,
   `context`. The example pipeline shows the mapping.
3. **Execution Auditor** — Settings → n8n API → create an API key → add it as an *n8n API*
   credential on the "Recent executions" node. Put your instance URL in the **Config** node so
   alerts link to the execution list. Pick the Slack credential and channel. Activate.

No credentials ship in these files (`REPLACE_ME` placeholders), and the example pipeline's URL
and spreadsheet id are placeholders too.

## The three ideas, in one paragraph each

**Alerts should be deduplicated at the source.** A workflow on a one-minute schedule against a
dead API generates sixty failed executions an hour. Sixty Slack pings is how people learn to
mute the channel, which is how the next real failure gets missed. The router fingerprints each
failure (workflow + node + first 80 chars of the message) in workflow static data and sends one
alert per fingerprint per 30 minutes with a running count.

**Run status is not outcome status.** n8n marks an execution successful when no node threw. It
cannot know that "append 0 rows" was not what you meant. The only thing that can know is a
check you write: *how many rows reached the write, and was that plausible?* Assert Outcome is
that check as a reusable sub-workflow, and it fails the *caller's* execution, so the Error
Router hears about it like any other failure.

**Look at the execution list so you don't have to.** Two shapes are visible from execution
metadata alone, without reading any workflow's data. A streak of consecutive errors that nobody
opened. And a bimodal duration split: if 700 "successful" runs each took 25 ms and the only runs
that took seconds all failed, the green runs were not doing the work — they were falling through
an IF before the expensive part. The auditor pulls the last 250 executions every hour and reports
both.

## What the kit does not do

- It cannot see inside a green execution's data. The auditor works from status and duration;
  the per-run item count is what Assert Outcome is for.
- The Error Router's dedup uses workflow static data, which only persists for production
  executions. Testing it with *Test workflow* will alert every time; trigger a real failure.
- The Auditor pulls 250 executions per run. On a busy instance that is a shorter window than an
  hour; lower the schedule interval or add a `workflowId` filter for the workflows you care most
  about.
- None of this replaces node-level *Retry On Fail* and *Timeout*. Those handle the transient
  failures; the kit handles the ones that are not transient and not loud.

Part of the [Levelbrook n8n portfolio](../README.md). Built by a senior engineer who runs
automations for clients and got tired of finding out on day three.
