# genkit

> Snapshot date: 2026-04-26. Upstream: <https://github.com/genkit-ai/genkit>

"**Open-source framework for building AI-powered apps in
JavaScript, Go, and Python.**" Genkit is Google / Firebase's
polyglot agent + flow framework: define typed flows, tools, and
prompts once in TypeScript / Go / Python, run them against any
supported model provider, and inspect every step (inputs, model
calls, tool calls, outputs) in a local Developer UI before
shipping the same flow to Cloud Functions, Cloud Run, or any
self-hosted Node / Go / Python service.

## 1. Install footprint

- Library + CLI per language:
  - JS/TS: `npm i genkit @genkit-ai/google-genai` (or other
    provider plugins) — primary surface, longest plugin list.
  - Go: `go get github.com/firebase/genkit/go` — same flow /
    tool / prompt primitives, idiomatic Go types.
  - Python: `pip install genkit` plus provider plugins — newer
    port, parity with the JS plugin matrix is in progress.
- Dev CLI: `npm i -g genkit-cli` installs `genkit` — `genkit
  start -- <run-cmd>` boots the Developer UI on `localhost:4000`
  with side-by-side inspector for traces, prompts, datasets,
  evals, and a flow runner.
- Deploy targets: Firebase Cloud Functions, Cloud Run, any
  Express / Hono / Fastify server, Next.js route handlers; Go
  flows compile to a normal binary and deploy anywhere Go does.

## 2. Repo + version + license

- Repo: <https://github.com/genkit-ai/genkit> (formerly
  `firebase/genkit`; the canonical org is now `genkit-ai`)
- Latest release tag: **v1.33.0** for the JS core and Vertex AI
  plugin (`vertexai-v1.33.0`, published 2026-04-24); each plugin
  ships its own semver tag from the same monorepo.
- License: **Apache-2.0** —
  <https://github.com/genkit-ai/genkit/blob/main/LICENSE>
- Default branch: `main`
- Languages: TypeScript (primary), Go, Python
- Stars: ~5.8k

## 3. Models supported

- Google: Gemini 2.x / 1.5 family via Google AI Studio
  (`@genkit-ai/google-genai`) and Vertex AI
  (`@genkit-ai/vertexai`), Imagen, Vertex embeddings.
- Third-party first-party-maintained plugins: OpenAI, Anthropic,
  Ollama, xAI, DeepSeek, Mistral, Cohere, Together, Groq, Fireworks
  via the `@genkit-ai/compat-oai` shim plus the official
  `genkitx-*` provider plugins.
- Embeddings + vector stores: Vertex Vector Search, Pinecone,
  Chroma, AstraDB, pgvector, plus a local file-backed dev-only
  store for fast iteration in the Developer UI.

## 4. Notable angle

**Polyglot flow framework with a real local developer surface for
trace inspection and eval.** Where most agent frameworks pick one
language and bolt observability on as a side feature, Genkit
treats the Developer UI as the primary loop: every flow run shows
each model call's request / response / token count / latency in a
tree, every tool call shows its typed inputs and outputs, prompts
are first-class versioned `.prompt` / `.dotprompt` files with a
side-by-side variant runner, and the same flow runs in JS, Go, or
Python with the same plugin schema. Compared with
[`langgraph`](../langgraph/) (Python-only graph orchestration) and
[`pydantic-ai`](../pydantic-ai/) (Python-only typed agents), Genkit
trades some Python ecosystem depth for cross-language portability
plus a deploy story tuned for Cloud Functions / Cloud Run; against
[`mastra`](../mastra/) (TS-only), Genkit adds Go and Python ports
and a more conservative provider plugin model. The catch is that
some plugins (Vertex / Google AI) are clearly first-class while
others lag a release behind the JS core — pin plugin versions per
flow, not just the core.

## 5. Last verified

2026-04-26 via `gh api repos/genkit-ai/genkit` and
`gh api repos/genkit-ai/genkit/releases` → core / vertexai
v1.33.0 published 2026-04-24.
