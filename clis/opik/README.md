# opik

> Snapshot date: 2026-04. Upstream: <https://github.com/comet-ml/opik>

"**Debug, evaluate, and monitor your LLM applications, RAG systems,
and agentic workflows.**"
`opik` is Comet's open-source LLM observability + evaluation platform.
The shape: a Python (and TS / Java / Ruby) SDK that emits structured
traces (`@track` decorator on any function turns it into a span),
plus a self-hostable React UI backed by ClickHouse / Postgres / Redis
for trace search, dataset curation, online + offline evaluation, prompt
versioning, and LLM-as-judge metric runs.

## 1. Install footprint

- Python SDK: `pip install opik` (Python ≥ 3.9). TS: `npm install
  opik`. Also Ruby + Java SDKs in the monorepo.
- Self-hosted backend: `opik server install` (one command, runs the
  full stack via Docker Compose: ClickHouse, Postgres, Redis, MinIO,
  Python backend, React frontend) or the Helm chart at
  `deployment/helm_chart/`.
- Hosted SaaS at `comet.com/opik` is the zero-install path; OSS
  self-host is fully feature-equivalent for tracing + eval (some
  enterprise SSO / RBAC features are commercial).
- CLI: `opik` (configure workspace, install local server, run health
  checks).

## 2. Repo + version + license

- Repo: <https://github.com/comet-ml/opik>
- Latest release: **2.0.14** (2026-04-24)
- License: **Apache-2.0** —
  <https://github.com/comet-ml/opik/blob/main/LICENSE>
- Default branch: `main`
- Language: Python (backend + SDK), TypeScript (frontend + JS SDK),
  Java, Ruby

## 3. Models supported

Provider-agnostic via instrumentation. First-class integrations:
OpenAI (Chat + Responses + Agents SDK), Anthropic, Gemini (AI Studio
+ Vertex), Bedrock, Azure OpenAI, Cohere, Mistral, Groq, DeepSeek,
Together, Fireworks, OpenRouter, Ollama, vLLM, llama-cpp, plus
LiteLLM-routed long tail. Framework integrations: LangChain,
LlamaIndex, [`crewai`](../crewai/), [`dspy`](../dspy/),
[`pydantic-ai`](../pydantic-ai/), [`smolagents`](../smolagents/),
Haystack, AG2, ADK. LLM-as-judge metrics use OpenAI / Anthropic /
Gemini / any OpenAI-compatible endpoint.

## 4. When to use it

- You want **traces + evals + prompt management + datasets in one
  self-hostable UI** instead of stitching Langfuse + Promptfoo +
  Argilla. Opik bundles all four under one Apache-2.0 install.
- You need **online evaluation rules** — define a metric (`Hallucination`,
  `AnswerRelevance`, `ContextPrecision`, custom Python), attach to a
  project, and every new trace gets scored automatically with the
  result back-annotated to the span; useful for "prod traffic
  regression-tests itself".
- You want **dataset → experiment → comparison** as the primary
  workflow: curate a `Dataset` from production traces, run an
  `Experiment` against N prompt or model variants, see the metric
  diff in the UI with statistical significance.
- You want a **fully OSS** observability stack (Apache-2.0, no ELv2
  trap) — contrast with [`arize-phoenix`](../arize-phoenix/) (Elastic
  License 2.0).

## 5. When NOT to use it

- You want **OTel-native spans landing in your existing Tempo /
  Jaeger / Datadog** — Opik uses its own ingest API, not OTLP. Pick
  [`openllmetry`](../openllmetry/) when the dashboards must be your
  current ones.
- You want **a single-binary local-only debugger** with no DB —
  Opik's self-host needs ClickHouse + Postgres + Redis. Reach for
  [`arize-phoenix`](../arize-phoenix/) (`pip install
  arize-phoenix; phoenix serve`) when local-loopback simplicity wins.
- You want **CI-shaped batch eval as a `pytest` plugin** — Opik
  supports pytest integration but the home idiom is `Experiment` runs
  surfaced in the UI; [`promptfoo`](../promptfoo/) and
  [`deepeval`](../deepeval/) feel more native to a CI-first workflow.
- You only need **prompt versioning** — overkill; a git-tracked
  prompt file plus [`promptfoo`](../promptfoo/) is lighter.

## 6. Closest alternatives

- [`arize-phoenix`](../arize-phoenix/) — closest peer; OpenInference /
  OTel-shaped spans, ELv2 license; Phoenix wins on OTel-native
  ingest, Opik wins on bundled prompt management + dataset curation +
  Apache-2.0.
- [`openllmetry`](../openllmetry/) — pure instrumentation, ships
  spans to any OTLP collector; pair with Opik when you want the
  Opik UI on top of OTel pipelines.
- [`ragas`](../ragas/) / [`deepeval`](../deepeval/) — eval libraries;
  Opik bundles equivalent metrics inside its own runner.
- [`promptfoo`](../promptfoo/) — CI-shaped prompt eval; lighter than
  Opik, no trace UI.
- [`mlflow`](../mlflow/) (if catalogued) — experiment tracking with
  recent LLM extensions; Opik is the LLM-native peer.

## 7. Repo health (snapshot)

- Very active: 2.0.x line shipped through April 2026, release cadence
  ~weekly (2.0.14 on 2026-04-24).
- Backed by Comet (commercial parent) — OSS / hosted feature parity is
  the explicit policy for tracing + eval.
- Multi-language SDKs (Python / TS / Java / Ruby) maintained in the
  monorepo with shared schema.
- Wide framework integration matrix is the day-1 bet — coverage as of
  2.0.14 includes LangChain, LlamaIndex, CrewAI, DSPy, smolagents,
  Pydantic-AI, OpenAI Agents, Haystack, ADK, Bedrock Agents.
