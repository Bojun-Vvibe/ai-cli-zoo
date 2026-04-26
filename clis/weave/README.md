# weave

- **Repo:** https://github.com/wandb/weave
- **Version:** `v0.52.37` (latest release)
- **License:** Apache-2.0 (`LICENSE`)
- **Language:** Python (TypeScript SDK in sibling package)
- **Install:** `pip install weave` then `import weave; weave.init("my-project")`

## One-line summary

Trace, evaluate, and version LLM/agent code from Weights & Biases — a
decorator-driven observability + eval toolkit that records every model call,
tool call, and prompt as a navigable run on the W&B platform (or a self-hosted
backend).

## What it does

Weave is library-first: drop `@weave.op` on any Python function and every
invocation becomes a checkpointed trace with inputs, outputs, latency, token
counts, and a stable content-addressable hash so identical calls dedupe.
Auto-instrumentation covers OpenAI, Anthropic, Gemini, LiteLLM, MistralAI,
Cohere, Groq, DSPy, LangChain, LlamaIndex, CrewAI, OpenAI Agents SDK, plus
NVIDIA NIM — every framework call lands as a nested span tree under the
top-level `op`.

The CLI surface is thin (`weave server` for local self-host) — the value is
the `Evaluation` API (run a dataset × model × scorer matrix, get a leaderboard
in the W&B UI), `Dataset` versioning (datasets are first-class artifacts that
diff like git commits), and `Model` subclasses (system prompt + decoding
params version together so you can re-run an old eval against today's model).

```python
import weave
from openai import OpenAI

weave.init("triage-bot")
client = OpenAI()

@weave.op
def classify(ticket: str) -> str:
    resp = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": f"Label: {ticket}"}],
    )
    return resp.choices[0].message.content

classify("My laptop won't boot")  # trace lands in W&B UI
```

## When to choose it

- You're already on Weights & Biases for ML experiment tracking and want
  one platform for both training runs and LLM app traces.
- You want **dataset + model + scorer leaderboards** as a first-class
  artifact, not a notebook of CSVs.
- You want auto-instrumentation across many frameworks (DSPy + CrewAI +
  raw OpenAI in the same trace tree) without writing OTel exporters.

## When NOT to choose it

- You want a fully self-hosted observability stack with no SaaS dependency
  by default — Weave's hosted backend is the canonical path; self-hosting
  exists but is heavier than `langfuse` / `langtrace` / `laminar`.
- You need an OTel-native stack that your existing Jaeger/Tempo/Grafana
  already consumes — Weave traces ship to W&B's proprietary backend, not
  OTLP.
- You're not in Python — the TypeScript SDK exists but is less mature.
