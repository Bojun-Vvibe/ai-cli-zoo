# open-webui

> Snapshot date: 2026-04. Upstream: <https://github.com/open-webui/open-webui>

A self-hostable, single-user-or-team **AI chat web UI** built around
the Ollama runtime but happy to talk to any OpenAI-compatible
endpoint. The bundle is a Python service plus a Svelte frontend, and
the install footprint is small enough that the canonical "get
started" path is one `pip install` and one process — no Compose
file, no database container, no separate vector store unless you
want one.

In the catalog it is the **default pick when "I have Ollama running
and I want a real UI in front of it"** is the actual problem. It
sits next to [`librechat`](../librechat/) (heavier, multi-user,
auth-first) and [`khoj`](../khoj/) (notes/RAG-first daemon with a
chat surface bolted on); `open-webui` chooses the *power-user single
host* trade-off — pipelines, Python functions, model presets,
per-model parameter tuning, RAG over uploaded docs, and a built-in
prompt library all expose themselves in the same UI.

## 1. Repo facts

- Repository: <https://github.com/open-webui/open-webui>
- Latest release: **v0.9.2** (2026-04-24)
- Primary language: Python (backend) + Svelte (frontend)
- License: **Open WebUI License** (BSD-3-Clause-derived with a
  branding-preservation clause for builds over 50 users) — see
  [`LICENSE`](https://github.com/open-webui/open-webui/blob/main/LICENSE)
  (blob SHA `99f39e7feff29c93342877adad2d5c15e707444c`). Permissive
  for personal / small-team self-hosting; check the branding clause
  before redistributing a fork or shipping it as a hosted product.

## 2. Why it matters

- **Local-first by default.** Point the UI at `http://localhost:11434`
  and you have a working chat over a local Llama / Qwen / Mistral
  model in under five minutes; cloud providers are an opt-in addition,
  not a prerequisite.
- **Pipelines are first-class.** The Pipelines feature lets you write
  a Python file that intercepts inbound messages, calls arbitrary
  tools, and rewrites the outbound response — the same surface that
  third-party plugins use, no fork required.
- **Document chat is built in.** Drop a PDF or a folder into a
  workspace, pick an embedding model (local or cloud), and the chat
  retrieves citations without you wiring a separate vector DB.

## 3. Install

The fast path is `pip` (Python ≥3.11):

```sh
pip install open-webui
open-webui serve
# UI at http://localhost:8080
```

Or via Docker if you want isolation:

```sh
docker run -d -p 3000:8080 \
  -v open-webui:/app/backend/data \
  --name open-webui \
  ghcr.io/open-webui/open-webui:main
```

## 4. Example invocation

The web UI is the primary surface, but the backend exposes an
OpenAI-compatible REST API once you mint an API key in
**Settings → Account**:

```sh
curl -X POST 'http://localhost:8080/api/chat/completions' \
  -H "Authorization: Bearer $OPEN_WEBUI_KEY" \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "llama3.1:8b",
    "messages": [{"role":"user","content":"hi"}],
    "stream": false
  }'
```

## 5. When to pick something else

- You need **team auth, OIDC/LDAP, per-user quotas** out of the box
  → [`librechat`](../librechat/).
- You want a **workflow canvas** with datasets and eval pipelines →
  [`dify`](../dify/) or [`langflow`](../langflow/).
- You want a **terminal** chat client →
  [`aichat`](../aichat/), [`oterm`](../oterm/), or
  [`mods`](../mods/).
- You want to **edit code on disk** from chat — wrong tool; see
  [`opencode`](../opencode/), [`aider`](../aider/), or
  [`claude-code`](../claude-code/).
