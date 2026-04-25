# trulens

> Snapshot date: 2026-04. Upstream: <https://github.com/truera/trulens>
> License file: <https://github.com/truera/trulens/blob/main/LICENSE>
> Pinned: `trulens-2.7.2` (2026-04-09). Active development on `main`;
> the 2.x line stabilised the OTel-native span model so pin the exact
> PyPI version (`trulens==2.7.2`, `trulens-core==2.7.2`,
> `trulens-providers-openai==2.7.2`) in production — feedback function
> internals continue to evolve and a re-eval against a moving version
> will silently change scores.

A **tracing + evaluation harness for LLM apps and agents** built on
OpenTelemetry. You wrap your app (function, LangChain chain,
LlamaIndex query engine, NeMo Guardrails rail, raw OpenAI / Anthropic
calls) in a `TruApp` / `TruChain` / `TruLlama` / `TruCustomApp`
recorder, attach **feedback functions** (groundedness, answer
relevance, context relevance, harmfulness, language match, ~20 more,
plus your own `Feedback(...)` over arbitrary span attributes), and
every invocation lands in a local SQLite (or your Postgres / Snowflake)
as a trace tree with feedback scores attached. The bundled
**TruLens Dashboard** (Streamlit) renders the leaderboard, per-record
trace viewer, and feedback drill-down.

The everyday surface is the Python SDK (`from trulens.core import
TruSession; session = TruSession()`); the dashboard is launched with
`run_dashboard(session)` or the `trulens` CLI shim.

## 1. Install footprint

- `pip install trulens` (umbrella) or pick a slimmer pair:
  `pip install trulens-core trulens-providers-openai
  trulens-dashboard`. `uv pip install` works.
- Python 3.9+. No daemon; the dashboard is an opt-in Streamlit
  process you start yourself.
- Workspace footprint: `default.sqlite` (the trace + feedback store)
  in cwd by default — point at Postgres / Snowflake via
  `TruSession(database_url=...)` for team installs. Add
  `default.sqlite` to `.gitignore`.
- Provider extras pull only what you need: `trulens-providers-openai`,
  `-anthropic`, `-bedrock`, `-litellm`, `-huggingface`,
  `-cortex` (Snowflake). Framework extras: `trulens-apps-langchain`,
  `-llamaindex`, `-nemo`.
- First dashboard run downloads a tiny set of static assets; everything
  else is local.

## 2. License

MIT (file is `LICENSE` at the repo root).

## 3. Models supported

Two distinct axes — same shape as [`deepeval`](../deepeval/) /
[`ragas`](../ragas/):

- **Models you are evaluating** — anything. TruLens records spans
  around your existing LLM calls (LangChain, LlamaIndex, raw SDK,
  custom function); the framework never speaks to your
  system-under-test directly.
- **Models the feedback functions use as judges** — configurable per
  feedback:
  - **OpenAI** — `gpt-4o`, `gpt-4o-mini`, `o1`, etc. via
    `trulens.providers.openai.OpenAI`.
  - **Anthropic** — Claude Sonnet / Opus / Haiku via the
    Bedrock provider or LiteLLM.
  - **Bedrock** — Claude / Llama / Titan / Mistral via
    `trulens.providers.bedrock.Bedrock`.
  - **LiteLLM** — long tail (Gemini / Mistral / Cohere / Groq /
    DeepSeek / OpenRouter / Together / Fireworks / Ollama / vLLM /
    llama.cpp) via `trulens.providers.litellm.LiteLLM`.
  - **Snowflake Cortex** — `trulens.providers.cortex.Cortex`
    (in-warehouse evaluation, no data egress).
  - **HuggingFace** — local classical NLP scorers (toxicity,
    language-detect, NLI groundedness) via
    `trulens.providers.huggingface.Huggingface` for a fully-offline
    feedback subset.

If you swap the judge provider, re-run your benchmark — judge choice
moves absolute scores meaningfully. Pin model name + version.

## 4. MCP support

**No first-party MCP transport at v2.7.2.** TruLens is an OTel-shaped
*observer* of LLM apps, not an agent or an MCP server. Two integration
shapes work today:

- Your MCP host (an agent) calls a TruLens-recorded function as a
  tool — instrumentation captures the call as a span like any other.
- An MCP server is one of the components inside your LLM app —
  pair TruLens with [`openllmetry`](../openllmetry/)'s
  `opentelemetry-instrumentation-mcp` to get MCP tool-call spans
  alongside the TruLens-emitted feedback spans (both land in OTel,
  TruLens-native span model normalises them in 2.x).

Roadmap mentions an MCP exporter so feedback scores become MCP
resources; not shipped at snapshot.

## 5. Sub-agent model

None at the framework level — TruLens is an observer, not an agent
runtime. Multi-agent apps work fine: each agent invocation becomes a
nested span tree, the dashboard's record viewer renders the parent /
child hierarchy, and feedback functions can scope to a sub-tree via
selectors (`Select.RecordCalls.planner.output`,
`Select.RecordCalls.worker.tools[0].args`).

For composing your own evaluators, `Feedback(...).on(...).aggregate(...)`
chains a provider call → a span selector → an aggregator (mean, max,
custom) — the deterministic scoring DAG counterpart to
[`deepeval`](../deepeval/)'s DAG metric.

## 6. Telemetry stance

**Off by default in 2.x.** The OSS SDK does not phone home; the
dashboard is a local Streamlit. Anonymous usage telemetry was opt-in
even in 1.x and was removed from the default install path in 2.0.

The judge provider obviously sees the prompts / outputs you score,
including production payloads in `actual_output` — for sensitive
evaluations route the feedback provider at a local model
(LiteLLM → Ollama / vLLM, or Snowflake Cortex for in-warehouse
scoring) so nothing leaves the boundary.

The TruLens database (`default.sqlite` or your configured Postgres /
Snowflake) holds full prompts, responses, retrieved contexts, and
feedback scores — treat it like any other LLM trace store and apply
your normal retention policy.

## 7. Prompt-cache strategy

**None at the TruLens layer.** Each feedback function call is a
fresh judge prompt. Two consequences:

- A re-run of the same eval over the same records costs the same
  money as the first run unless you snapshot scores and re-load
  rather than re-compute. The `TruSession.get_records_and_feedback()`
  call returns a DataFrame you can pickle.
- OpenAI / Anthropic provider-side prompt caching does fire when
  feedback prompts are stable across records (TruLens uses fixed
  rubric templates per feedback function), so amortised cost runs
  ~30–50% under naive token counts on real workloads.

For expensive nightly evals, use `TruSession.run_feedback_functions(
record, feedback_functions, mode="deferred")` to enqueue scoring,
then a worker process drains the queue with bounded concurrency —
this lets a 10k-record sweep finish overnight under a controlled
provider rate-limit budget instead of saturating it in the first
five minutes.

## 8. Hot keybinds

No TUI. Three surfaces:

- **Python SDK**:
  ```python
  from trulens.core import TruSession, Feedback, Select
  from trulens.providers.openai import OpenAI
  from trulens.apps.custom import TruCustomApp

  provider = OpenAI(model_engine="gpt-4o-mini")
  f_groundedness = (
      Feedback(provider.groundedness_measure_with_cot_reasons,
               name="Groundedness")
      .on(Select.RecordCalls.retrieve.rets[:].page_content.collect())
      .on_output()
  )
  f_relevance = Feedback(provider.relevance, name="Answer Relevance"
                         ).on_input_output()

  tru_app = TruCustomApp(my_rag_app,
                         app_name="rag-v3",
                         feedbacks=[f_groundedness, f_relevance])
  with tru_app as recording:
      my_rag_app.query("Who wrote Pride and Prejudice?")
  ```
- **Dashboard**:
  ```python
  from trulens.dashboard import run_dashboard
  run_dashboard(session, port=8484)
  ```
  Tabs: Leaderboard (apps × feedbacks, mean / median / latency / cost),
  Records (per-record trace viewer with span tree + per-feedback
  drill-down + reasoning chain for CoT feedbacks), Compare (A/B
  two app versions on the same input set with statistical sig).
- **CLI**: `trulens` shim mostly delegates to the dashboard
  (`trulens run-dashboard --port 8484 --database-url
  sqlite:///default.sqlite`). No `trulens test run` equivalent —
  evaluation lives in your Python script / pytest.

Programmatic batch eval pattern:

```python
from trulens.core import TruSession
session = TruSession()
records, feedback_cols = session.get_records_and_feedback(
    app_ids=["rag-v3"])
records[["app_id", "input", "Groundedness", "Answer Relevance",
         "latency", "total_cost"]].to_csv("rag-v3-eval.csv")
```

## 9. Killer feature, weakness, when to choose

**Killer feature.** **OTel-native span model + bundled local
dashboard, both in one Apache-style install**. The 2.x rewrite moved
the trace + feedback store onto the OpenTelemetry data model
(`SpanAttributes` for inputs / outputs / retrieved contexts / tool
calls / feedback scores), so a TruLens-recorded run composes cleanly
with any other OTel pipeline you already have — and the bundled
Streamlit dashboard means you don't need a separate platform deploy
to *see* the data. The feedback-function catalog is broad
(groundedness, answer-relevance, context-relevance, comprehensiveness,
harmfulness, language-match, sentiment, controversiality, criminality,
maliciousness, insensitivity, summarisation quality, conciseness,
coherence, correctness, custom Pydantic-typed) and the
`Feedback(...).on(Select...)` selector grammar lets you score *any*
sub-tree of *any* recorded run without modifying the app.

**Weakness.** **Smaller community than the eval-tool incumbents and
heavier conceptual lift than the pytest-shaped runners**. The
`Select.RecordCalls.<path>` selector grammar is powerful but takes
real time to internalise — there is a learning curve before the
"oh, *that's* how I score the retriever's output" moment lands.
Feedback re-runs are not memoised (same caveat as
[`deepeval`](../deepeval/) / [`ragas`](../ragas/)) so cost discipline
matters. Snowflake's acquisition of TruEra reorganised the project;
the 2.x rewrite landed cleanly but some 1.x tutorials and blog posts
still reference removed APIs — pin to 2.x docs and cross-check
imports against the current PyPI version.

**When to choose.**

- You have an LLM app (RAG, chatbot, agent) and you want **traces +
  feedback scores + a local dashboard** in one install, without
  standing up a separate observability platform.
- You need to **score arbitrary sub-spans** of a complex pipeline
  (e.g. "rate the retriever's context relevance independently from
  the synthesizer's groundedness") — the selector grammar is built
  for exactly this.
- You want **OTel-native trace data** that composes with the rest
  of your observability stack, not a vendor-specific span shape.
- You operate inside Snowflake and want **in-warehouse feedback
  evaluation** via Cortex with no data egress — first-class provider.

**When not to choose.** You want a **pytest-shaped CI runner** that
drops into existing test infrastructure with no learning curve —
[`deepeval`](../deepeval/) is closer to that shape (one
`deepeval test run tests/` and JUnit XML drops into GitHub Actions).
You only need the **canonical RAG metrics** (faithfulness,
context-precision / recall) without a dashboard or trace store —
[`ragas`](../ragas/) is the lightest pick and the one papers cite.
You want a **prompt-comparison matrix** across providers with a
diff viewer and no eval runtime in your code — [`promptfoo`](../promptfoo/)
is closer. You want a **bundled UI with traces + evals + prompt
management + datasets** that does not require you to write feedback
selectors — [`opik`](../opik/) ships that exact stack under
Apache-2.0. You need **OTel ingest into Tempo / Jaeger / Datadog
without a TruLens-specific store** — [`arize-phoenix`](../arize-phoenix/)
or [`openllmetry`](../openllmetry/) sit closer to that boundary.
