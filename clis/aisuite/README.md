# aisuite

> Snapshot date: 2026-04. Upstream: <https://github.com/andrewyng/aisuite>

"**Simple, unified interface to multiple Generative AI providers.**"
`aisuite` is a thin Python adapter layer (initiated by Andrew Ng's
group) that exposes a single `client.chat.completions.create()` call
shaped like the OpenAI SDK and dispatches it to OpenAI, Anthropic,
Google (Gemini / Vertex), AWS Bedrock, Azure OpenAI, Groq, Mistral,
Cohere, DeepSeek, Hugging Face, Ollama, OpenRouter, SambaNova,
Together, xAI, watsonx, etc., picking provider via a
`"<provider>:<model>"` string. Goal: write `chat()` once, swap
providers by changing the model string — no per-vendor SDK in your
business logic.

## 1. Install footprint

- `pip install aisuite` (Python ≥ 3.10). Core install is intentionally
  lean — provider SDKs are optional extras: `pip install
  'aisuite[openai]'`, `'aisuite[anthropic]'`, `'aisuite[google]'`,
  `'aisuite[aws]'`, `'aisuite[mistral]'`, `'aisuite[groq]'`,
  `'aisuite[huggingface]'`, `'aisuite[ollama]'`, `'aisuite[all]'`.
- Library-first; not a dedicated CLI binary. Use as
  `import aisuite as ai; client = ai.Client()` from any Python REPL,
  notebook, or script — including ad-hoc one-liners via `python -c`.
- Auth via per-provider env vars (`OPENAI_API_KEY`,
  `ANTHROPIC_API_KEY`, `GOOGLE_PROJECT_ID`, `AWS_ACCESS_KEY_ID`,
  `GROQ_API_KEY`, …) or a `provider_configs={...}` dict on the
  `Client`.

## 2. Repo + version + license

- Repo: <https://github.com/andrewyng/aisuite>
- Latest release: **v0.1.7** (2024-12-26)
- License: **MIT** —
  <https://github.com/andrewyng/aisuite/blob/main/LICENSE>
- Default branch: `main`
- Language: Python (Jupyter notebooks for examples)

## 3. Models supported

Anything reachable through the wrapped provider SDKs. First-class
adapters include OpenAI (chat completions), Anthropic (messages),
Google (Gemini AI Studio + Vertex), AWS Bedrock (Anthropic / Llama /
Mistral / Titan), Azure OpenAI, Mistral La Plateforme, Groq, Cohere,
DeepSeek, Hugging Face Inference, Ollama, OpenRouter, SambaNova,
Together, xAI Grok, watsonx, Fireworks. Selection is purely lexical:
`model="openai:gpt-4o"`, `model="anthropic:claude-3-5-sonnet"`,
`model="ollama:llama3.1"`, `model="groq:llama-3.1-70b-versatile"`.

## 4. When to use it

- You are writing **provider-agnostic glue** — a CLI tool, a notebook
  demo, a teaching example, an evaluation harness — and you want to
  benchmark "same prompt across N providers" without authoring N SDK
  call sites. The `model` string is the only thing that changes.
- You want a **dependency-light entry point** for students /
  prototypes: `pip install aisuite` plus one provider extra, no
  framework, no agent loop, no router.
- You are pairing with [`promptfoo`](../promptfoo/) /
  [`deepeval`](../deepeval/) /
  [`ragas`](../ragas/) and need a uniform completion call inside the
  evaluator code so the eval logic does not branch on provider.

## 5. When NOT to use it

- You need **streaming + tool-calling + structured outputs +
  retries** with provider-specific feature parity — the surface here
  is deliberately the OpenAI chat-completions subset, and provider
  quirks (Anthropic `tool_use` blocks, Gemini parts arrays, Bedrock
  invoke shapes) are flattened away. Reach for
  [`instructor`](../instructor/) (structured outputs +
  per-provider mode) or [`magentic`](../magentic/) instead when you
  need full feature surface.
- You want a **hosted proxy with routing / cost caps / fallback /
  rate-limit pooling** — `aisuite` is in-process Python; the proxy
  job is owned by [`claude-code-router`](../claude-code-router/) and
  by LiteLLM-style HTTP gateways.
- You want **agentic tool use, planners, multi-agent
  orchestration** — out of scope; pick
  [`crewai`](../crewai/) / [`smolagents`](../smolagents/) /
  [`pydantic-ai`](../pydantic-ai/) /
  [`agno`](../agno/) instead.
- You need a **TypeScript / Go / Rust** path — Python only at the
  snapshot date.

## 6. Closest alternatives

- [`instructor`](../instructor/) — same provider-fanout aesthetic
  but with Pydantic structured outputs as the headline feature.
- [`magentic`](../magentic/) — decorator-driven, structured-output
  Python wrapper across many providers.
- [`pydantic-ai`](../pydantic-ai/) — Pydantic-team agent framework
  with the same multi-provider story plus tools, deps, and graphs.
- [`llm`](../llm/) — Simon Willison's CLI / library with a plugin
  ecosystem for providers; CLI-first vs aisuite's library-first.
- LiteLLM (not in catalog as a standalone CLI here) — the bigger,
  proxy-shaped sibling: HTTP gateway, routing, cost tracking,
  fallback, observability.

## 7. Repo health (snapshot)

- Last release: **v0.1.7** on 2024-12-26 — releases are infrequent;
  treat as a pedagogical / glue layer rather than a fast-moving
  framework.
- `main` continues to land provider adapters and example notebooks.
- Pre-1.0 — API surface (`Client`, `chat.completions.create`,
  `model="provider:model"`) has been stable since 0.1.x but check
  the changelog before a long-lived integration.
