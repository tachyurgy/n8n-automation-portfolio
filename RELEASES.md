# Releases

## 2026-07-11 — Initial launch: automation portfolio + 3 workflows
- **What deployed:** Portfolio site live at **https://automation.levelbrook.com** (Cloudflare Pages project `levelbrook-automation`, CNAME proxied). Public repo live at **https://github.com/tachyurgy/n8n-automation-portfolio**.
- **Changed:**
  - 3 real, importable n8n workflows: `01-rag-knowledge-agent` (30 nodes), `02-lead-enrichment-scoring` (23), `03-content-repurposing` (23) — each with a Problem→Workflow→Outcome case-study README. Node types mirrored from real n8n templates so they import.
  - Portfolio site: editorial design, 3 case studies, services, contact (levelbrookteam@gmail.com). No rates named.
  - MIT license, brand-authored commits (Levelbrook Consulting).
- **How:** `wrangler pages deploy` to project `levelbrook-automation`; custom domain via Pages domains API + CNAME `automation`→pages.dev (proxied). Repo via `gh repo create tachyurgy/... --public --push`, branch renamed master→main.
- **Verified:** automation.levelbrook.com → HTTP 200 (TLS issued); pages.dev → 200; repo → 200; all 3 workflow.json validate via `python3 json.load`; forbidden-identity grep clean; site screenshot-checked (no overflow) via cached Chromium.
