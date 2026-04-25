# logfire

> Snapshot date: 2026-04. Upstream: <https://github.com/pydantic/logfire>

"**AI observability platform for production LLM and agent systems.**"
Logfire is the Pydantic team's OpenTelemetry-native tracing /
logging / metrics SDK plus a CLI (`logfire`) that authenticates
projects, prints live tail, runs the local OTLP collector, and
manages the hosted backend. The Python SDK auto-instruments the
LLM and agent stacks Pydantic users already run — `pydantic-ai`,
OpenAI, Anthropic, LiteLLM, LangChain, LlamaIndex, FastAPI, HTTPX,
SQLAlchemy, asyncio — so an existing app gets traces, prompt /
completion capture, token-cost rollups, and structured logs by
adding two lines.

## 1. Install footprint

- `pip install logfire` (SDK + `logfire` CLI). Per-integration
  extras: `logfire[fastapi]`, `logfire[httpx]`, `logfire[openai]`,
  `logfire[anthropic]`, `logfire[asyncpg]`, `logfire[redis]`,
  `logfire[django]`, etc.
- CLI: `logfire auth`, `logfire projects new`, `logfire projects
  use`, `logfire whoami`, `logfire info`, `logfire inspect`,
  `logfire run` (wrap a process), `logfire clean`.
- Pure OTLP under the hood — also ships to any OTel collector
  (Jaeger / Tempo / Honeycomb / Datadog / Grafana Cloud) via
  `OTEL_EXPORTER_OTLP_ENDPOINT`; the hosted backend at
  `logfire.pydantic.dev` is optional, not required.
- Python ≥ 3.8.

## 2. Repo + version + license

- Repo: <https://github.com/pydantic/logfire>
- Latest release: **v4.32.1** (2026-04-15)
- License: **MIT** —
  <https://github.com/pydantic/logfire/blob/main/LICENSE>
- HEAD SHA: `765a602d91aa7fd0c49c082f154a2a4b8920df78`
- Default branch: `main`
- Language: Python

## 3. Models supported

Provider-agnostic — instruments calls into OpenAI, Anthropic,
Google, Mistral, Cohere, Groq, Ollama, any LiteLLM-routed
backend, plus whatever `pydantic-ai` / LangChain / LlamaIndex are
calling. Captures model name, prompt, completion, token counts,
tool calls, latency, errors as OTel span attributes.

## 4. Simple usage

```bash
# 1. one-time auth (browser flow) + project create
logfire auth
logfire projects new my-llm-app
logfire projects use my-llm-app
```

```python
# 2. two lines in your app
import logfire
logfire.configure()                       # reads creds from ~/.logfire
logfire.instrument_openai()               # auto-trace every OpenAI call
logfire.instrument_pydantic_ai()          # auto-trace every Agent run
logfire.instrument_httpx()                # plus outbound HTTP

from openai import OpenAI
OpenAI().chat.completions.create(
    model="gpt-4o-mini",
    messages=[{"role":"user","content":"hi"}],
)
# → span in the dashboard with prompt, completion, tokens, cost, latency
```

```python
# 3. structured logs co-located with traces
with logfire.span("planning step", user_id=42):
    logfire.info("retrieved {n} candidates", n=len(cands))
```

## 5. Why it's interesting

- **OpenTelemetry under the hood, not a proprietary protocol** —
  same SDK can dual-export to the hosted Logfire backend and your
  existing OTel collector; if you walk away from the SaaS the
  instrumentation stays.
- **Python f-string-style structured logging** (`logfire.info("got
  {n} hits", n=len(hits))`) compiles to a templated message + typed
  attributes in the OTel span, so you get human-readable lines
  *and* queryable structured fields from one call site — no
  separate `extra={}` dict.
- **Auto-instrumentation that knows the LLM shape**, not just
  HTTP: prompt + completion + tool_calls + token counts + model
  name land as first-class span attributes for OpenAI / Anthropic
  / LiteLLM / `pydantic-ai`, so trace search like "all spans where
  model = gpt-4o and prompt_tokens > 8000" works without manual
  wrapping.
- **Pydantic-native validation tracing** — `pydantic-ai` Agents,
  `BaseModel` validation errors, and `TypeAdapter` calls all show
  up with the offending field path on the span, closing the loop
  between "the model returned bad JSON" and "this validator
  rejected it on this field".

## 6. Caveats

- The hosted backend is a paid SaaS beyond a generous free tier;
  pure-OSS users should configure a self-hosted OTel collector
  endpoint.
- Auto-instrumentation captures prompts and completions by
  default — review `capture_args` / scrubbing config before
  pointing it at PII-bearing workloads.
- SDK is Python-only today; for Node / Go / Rust services use any
  standard OTel SDK and ship to the same collector.
