# mlflow

> Snapshot date: 2026-04. Upstream: <https://github.com/mlflow/mlflow>
> License file: <https://github.com/mlflow/mlflow/blob/master/LICENSE.txt>
> Pinned: `v3.11.1` (2026-04-08, Python package + tracking server).
> Default branch is `master`. Install as `pip install mlflow` (full)
> or `pip install mlflow-skinny` (no SQL/server, ~10× smaller).

A **lifecycle platform for ML and now GenAI**: experiment tracking,
model registry, deployment, and — since the 2.x → 3.x line — a
first-class **GenAI tracing + evaluation** surface (`mlflow.genai`)
that records prompts, completions, retrieval spans, and tool calls
to the tracking server with the same schema as a classic ML run.
The contrast point against [`langfuse`](../langfuse/) and
[`helicone`](../helicone/) is "the tracing UI is a tab inside a
full MLOps stack you probably already run" rather than a dedicated
LLM observability product.

## 1. Install footprint

- **Library**: `pip install mlflow` (~250 MB with `sqlalchemy`,
  `alembic`, `flask`, `gunicorn`, `pyarrow`, `numpy`, `pandas`,
  `scikit-learn`); `pip install mlflow-skinny` (~25 MB) for client-
  only / serverless use.
- **Tracking server**: `mlflow server --backend-store-uri
  postgresql://... --default-artifact-root s3://bucket/mlflow
  --host 0.0.0.0 --port 5000` — Flask + gunicorn UI, REST API,
  artifact proxy.
- **Local dev**: `mlflow ui` (SQLite + local `./mlruns` directory)
  is the zero-config shape; everything writes to the cwd.
- **Model serving**: `mlflow models serve -m models:/my-model/
  Production -p 1234` spins up a FastAPI predictor; `mlflow models
  build-docker` produces a deployable image.

## 2. License

Apache-2.0 (file is `LICENSE.txt` at the repo root,
<https://github.com/mlflow/mlflow/blob/master/LICENSE.txt>). The
OSS server, client SDKs (Python / Java / R / JS), and the GenAI
modules are all Apache-2.0. The Databricks-managed MLflow is the
commercial hosted offering; the OSS server has no feature gating.

## 3. Models supported

For classic ML, MLflow's "flavors" cover sklearn / xgboost /
lightgbm / pytorch / tensorflow / onnx / spark / catboost /
statsmodels / prophet / transformers and more. For GenAI, the
`mlflow.genai` module auto-instruments the major SDKs (OpenAI,
Anthropic, LangChain, LlamaIndex, DSPy, AutoGen, LiteLLM,
LangGraph, OpenAI-Agents) so any call through those clients
becomes a span tree in the tracking UI. The `mlflow.genai.evaluate`
API takes an arbitrary callable + a dataset + a list of judges
(`faithfulness`, `relevance`, `correctness`, custom LLM-as-judge)
and emits a run with per-row scores.

## 4. MCP support

No first-party MCP server. The tracking REST API + Python client
are the integration surface; for "let an agent query past runs"
the obvious wiring is a thin MCP wrapper around `mlflow.search_runs`
and `mlflow.genai.search_traces`. The autologging side is the
inverse — agents wired through OpenAI / Anthropic / LangChain
clients land their traces in MLflow without code changes once
`mlflow.openai.autolog()` (etc.) is called once at process start.

## 5. Sub-agent model

Not a sub-agent framework. MLflow is the substrate that *records*
multi-agent runs — span hierarchy, tool-call attribution,
retrieval span types, and eval scores per leaf. For graphs,
LangGraph / AutoGen / OpenAI-Agents / DSPy autologging emits one
trace per top-level invocation with nested spans for each agent
hop, so post-hoc analysis ("which agent regressed in p95
latency?") is a `search_traces` query.

## 6. Telemetry stance

The tracking server is **the telemetry sink** — it is opt-in by
construction (you point `MLFLOW_TRACKING_URI` at it). The library
itself does not phone home; usage analytics for the OSS server are
off by default. For prompt + completion logging, autolog defaults
err on the side of capturing inputs/outputs — set
`mlflow.openai.autolog(log_inputs=False, log_outputs=False)` if
prompts contain secrets, or use the new `mlflow.genai.set_redactor`
hook (3.x) for column-level masking before write.

## 7. Prompt-cache strategy

Not a cache. Prompt caching is provider-side; MLflow's role is to
record the cache-hit metadata when the provider returns it (e.g.
Anthropic's `cache_read_input_tokens`) so you can chart cache hit
rate per prompt template over time. Pair with
[`gptcache`](../gptcache/) for an actual cache layer.

## 8. Hot keybinds

No TUI; the everyday surface is the web UI at `http://localhost:5000`
(experiments, runs, traces, models tabs) plus the Python client.

```python
import mlflow
import openai

mlflow.set_tracking_uri("http://localhost:5000")
mlflow.set_experiment("rag-eval")
mlflow.openai.autolog()  # captures every chat.completions call as a trace

with mlflow.start_run(run_name="prompt-v3"):
    mlflow.log_param("prompt_template", "v3")
    client = openai.OpenAI()
    out = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": "Summarise: ..."}],
    )
    mlflow.log_metric("response_tokens", out.usage.completion_tokens)

# Eval shape
import pandas as pd
results = mlflow.genai.evaluate(
    data=pd.DataFrame({"inputs": [...], "ground_truth": [...]}),
    predict_fn=lambda x: client.chat.completions.create(
        model="gpt-4o-mini", messages=[{"role": "user", "content": x}]
    ).choices[0].message.content,
    scorers=[
        mlflow.genai.scorers.Correctness(),
        mlflow.genai.scorers.RelevanceToQuery(),
    ],
)
```

## 9. Killer feature, weakness, when to choose

**Killer feature.** **One platform for classic-ML lifecycle and
GenAI tracing/eval, with the GenAI surface (`mlflow.genai`)
auto-instrumenting every major SDK** (OpenAI / Anthropic /
LangChain / LlamaIndex / DSPy / AutoGen / LangGraph / LiteLLM /
OpenAI-Agents). If you already run MLflow for sklearn /
transformers, the LLM observability tab arrives for free — same
runs, same registry, same RBAC, same artifact store.

**Weakness.** The footprint is heavy: full `mlflow` install pulls
~250 MB and the tracking server expects a real DB + artifact store
to be useful at team scale. The UX is built for "experiments and
runs", so per-trace exploration feels less first-class than
[`langfuse`](../langfuse/) or [`arize-phoenix`](../arize-phoenix/),
which were designed LLM-first. Eval scorers are competent but
shallower than [`deepeval`](../deepeval/)'s metric catalogue.

**When to choose.**

- You **already run MLflow** for classic ML and want LLM tracing /
  eval inside the same stack instead of standing up a second
  observability product.
- You need a **model registry + GenAI tracing in one server** with
  the same RBAC, audit log, and artifact store.
- You want **autologging for the long tail of GenAI SDKs** (DSPy,
  LangGraph, OpenAI-Agents) without writing per-SDK glue.

**When not to choose.** You want a **purpose-built LLM
observability product** with cohort analysis, eval datasets as a
first-class object, and prompt management — [`langfuse`](../langfuse/)
and [`helicone`](../helicone/) are leaner. You want a **deep
GenAI-eval metric catalogue** — [`deepeval`](../deepeval/) and
[`ragas`](../ragas/) ship more out of the box. You want **zero-
infra hosted tracing** — Helicone / Langfuse Cloud are closer to
that shape; OSS MLflow is a server you operate.
