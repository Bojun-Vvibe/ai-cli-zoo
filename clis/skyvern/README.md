# skyvern

> Snapshot date: 2026-04. Upstream: <https://github.com/Skyvern-AI/skyvern>

"**Automate browser-based workflows using LLMs and computer
vision.**" Skyvern is a browser-automation agent that takes a
plain-English goal plus a starting URL and drives a real Chromium
session (Playwright under the hood) to completion — fill the form,
download the invoice, complete the checkout — using a vision +
DOM hybrid prompt instead of brittle XPath / CSS selector scripts.
It is shipped as a long-running FastAPI service with a Postgres
backend, a React UI, and a `skyvern` CLI that wraps service
boot, task submission, and run inspection. The pitch is that one
prompt + one URL replaces the typical RPA pipeline (Selenium
fixtures, per-site selector maintenance, retry logic), because
the model re-derives the page model on every step from a fresh
annotated screenshot.

## 1. Install footprint

- `pip install skyvern` — pulls FastAPI, Playwright, SQLAlchemy,
  Alembic, LiteLLM, Litestar; `playwright install chromium` adds
  ~400 MB of browser bytes.
- `skyvern init` runs the interactive setup wizard: provisions
  Postgres (local or via the bundled `docker-compose.yml`),
  generates `.env`, picks the LLM provider, and seeds the schema
  via Alembic.
- `skyvern run server` boots the FastAPI service on `:8000`;
  `skyvern run ui` boots the React frontend on `:8080`.
- Python ≥ 3.11. Ships a Helm chart and a multi-arch Docker image
  for cluster deployments; the local-laptop path is the same
  CLI + a SQLite alternative for dev.

## 2. Repo + version + license

- Repo: <https://github.com/Skyvern-AI/skyvern>
- Latest release: **v1.0.31** (2026-04-14)
- License: **AGPL-3.0** —
  <https://github.com/Skyvern-AI/skyvern/blob/main/LICENSE>
- HEAD SHA: `a6203ef52bbe248a9bcbe1d9a1dc480802767595`
- Default branch: `main`
- Language: Python (+ TypeScript UI)

## 3. Models supported

Anything LiteLLM speaks, configured per-task or per-deployment:
OpenAI (GPT-4.1 / 4o / o-series), Anthropic Claude (Sonnet /
Opus / Haiku, including the computer-use family), Gemini (AI
Studio + Vertex), Bedrock, Azure-hosted endpoints, Groq,
DeepSeek, Ollama / vLLM / any OpenAI-compatible base URL.
Vision-capable models are strongly recommended — the agent loop
feeds the model an annotated screenshot every step, and
text-only models lose accuracy on rendering-heavy pages
(modals, sticky headers, dynamic ads).

## 4. Simple usage

```bash
pip install skyvern
playwright install chromium
skyvern init                 # interactive wizard, writes .env + DB
skyvern run server &         # FastAPI on :8000
skyvern run ui &             # React on :8080

# Submit a task from the CLI (JSON over the local API)
skyvern tasks create \
  --url "https://news.ycombinator.com/" \
  --goal "Open the top story, scroll to the comments, and return the first 3 comment authors as a JSON list"

# List runs and stream logs
skyvern tasks list
skyvern tasks logs <task-id> --follow
```

The same task can also be submitted via `POST /api/v1/tasks` for
service callers, or as a Python `from skyvern import Skyvern;
await Skyvern().run_task(...)` coroutine for in-process embedding.

## 5. Why it's interesting

- **Vision + DOM hybrid prompt** — every step ships the model
  both an annotated screenshot (numbered overlays on interactive
  elements) and the matching DOM string with the same numbering,
  so the model can reason about layout *and* unique element ids
  without trusting either alone; recovers from late-loading
  modals, A/B variants, and locale shifts that break XPath-only
  scripts.
- **Workflow blocks compose multi-page tasks** — `Workflow`
  YAML chains tasks, loops over a CSV of inputs, branches on
  extracted JSON, and feeds outputs forward (downloaded file →
  next task's upload field), so "open 200 supplier portals and
  download last month's invoice" is one workflow, not 200
  scripts.
- **Real artifact capture** — every step persists screenshot +
  DOM + action + LLM messages to the artifact store; failed runs
  are reproducible and inspectable in the UI without re-driving
  the browser.
- **Production-shaped** from day one: AGPL service with a Helm
  chart, Postgres state store, Alembic migrations, multi-tenant
  org scoping, webhook callbacks, S3-compatible artifact backend
  — closer to "deploy this in your VPC" than "run a notebook".

## 6. Caveats

- **AGPL-3.0** is the catch — embedding the service in a hosted
  product triggers the network-use clause; commercial use through
  Skyvern's own cloud (`skyvern.com`) avoids this, self-hosting
  with closed downstream products does not.
- Vision-model spend dominates the bill: a 30-step e-commerce
  checkout is ~30 image-conditioned chat calls, each ~50–150 KB
  of image plus ~5–20 KB of DOM; budget accordingly versus
  cheaper headless-browser RPA on stable sites.
- Long-running browser sessions are stateful — sites with strict
  bot detection (Cloudflare Turnstile, hCaptcha, advanced
  fingerprinting) still block; Skyvern's stealth defaults help
  but are not magic.
- Heaviest dependency surface in the catalog's browser-agent
  cluster (Playwright + Postgres + FastAPI + React) — for a
  single-shot "click through this page once" task, a code-first
  agent like [`browser-use`](../browser-use/) is lighter.
