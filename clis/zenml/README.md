# zenml

> Snapshot date: 2026-04. Upstream: <https://github.com/zenml-io/zenml>

An open-source **MLOps + LLMOps framework** that models ML and
LLM workflows as typed Python `@step` functions composed into
`@pipeline` DAGs, then runs the same pipeline locally for dev,
on Kubernetes for production, on Vertex / SageMaker / Kubeflow /
Tekton / Airflow / Skypilot for cloud, with **artifact
versioning, lineage tracking, and a stack abstraction** that
swaps the orchestrator + artifact store + container registry +
secrets backend without touching pipeline code. Since 2024 it has
extended into LLMOps territory: prompt versioning, RAG pipeline
templates, agent evaluation runs, and a unified dashboard tracking
both classical ML training runs and LLM-pipeline runs side by side.

It is the catalog's reference for **pipeline-as-code MLOps with a
swappable execution stack and a unified ML+LLM dashboard** — the
"runs in your cloud, not in their SaaS" alternative to managed
MLOps platforms.

## 1. Install footprint

- `pip install "zenml[server]"` (Python 3.9+).
- Pulls `pydantic`, `sqlalchemy`, `fastapi`, `click`, `alembic`,
  ~30 transitive deps. ~180 MB venv with the server extra.
- `zenml init` scaffolds a project, `zenml login --local` boots
  a local FastAPI dashboard on `:8237` backed by SQLite.
- Models: framework-agnostic — pipelines call whatever ML
  framework (`scikit-learn`, `pytorch`, `transformers`,
  `lightning`, `xgboost`) or LLM SDK (`openai`, `anthropic`,
  `litellm`, `langchain`, `llama-index`) you import; ZenML
  doesn't wrap the model call, it tracks the artefacts around it.
- Stack components are pip-installed integrations:
  `zenml integration install kubernetes mlflow s3 gcp aws`.

## 2. Repo, version, license

- Repo: <https://github.com/zenml-io/zenml>
- Version checked: **0.94.3** (latest tagged release).
- HEAD pinned at this snapshot:
  `3705567b463b30bdac0b7114e81067505419c89e`.
- License: Apache-2.0. License file at
  [`LICENSE`](https://github.com/zenml-io/zenml/blob/main/LICENSE).

## 3. What it actually does

A pipeline is decorated Python:

```python
from zenml import pipeline, step

@step
def load_data() -> pd.DataFrame: ...

@step
def train(df: pd.DataFrame) -> ClassifierMixin: ...

@step
def evaluate(model: ClassifierMixin, df: pd.DataFrame) -> dict: ...

@pipeline
def training_pipeline():
    df = load_data()
    model = train(df)
    metrics = evaluate(model, df)

training_pipeline()
```

Running this:

1. Each `@step` return value is **automatically materialized**
   to the configured artefact store (local fs / S3 / GCS / Azure
   Blob / MinIO) with a content-addressable URI.
2. The DAG is dispatched to the configured **orchestrator**
   (local thread pool, Docker, Kubernetes, Kubeflow, Vertex,
   SageMaker, Airflow, Tekton, Skypilot, Databricks).
3. **Lineage** is recorded: every step run links to the artefact
   versions it consumed and produced; the dashboard renders the
   DAG with click-through to inputs / outputs / logs / source code.
4. The **stack** (orchestrator + artefact store + container
   registry + secrets manager + experiment tracker + model
   registry + alerter) is swapped via `zenml stack set <name>` —
   the same pipeline runs locally for dev, on K8s for staging,
   on Vertex for prod with zero code changes.

The 2024+ LLMOps surface adds: `Prompt` artefact type with
versioning, RAG-pipeline templates (load → chunk → embed → index
→ query → eval), `EvaluationRun` for LLM evals tracked alongside
classical training runs, integrations for `langchain` /
`llama-index` / `litellm` / `openai` so prompts + retrieved
contexts + responses are first-class artefacts in the lineage
graph.

## 4. MCP support

None first-party as of 0.94.3. ZenML runs at the pipeline-orchestration
layer above any agent runtime; if a `@step` calls an MCP-aware
agent ([`pydantic-ai`](../pydantic-ai/),
[`openai-agents-python`](../openai-agents-python/)), the MCP
calls happen inside the step and ZenML tracks only the step's
inputs/outputs/artefacts.

## 5. Sub-agent model

None — ZenML is an orchestrator for typed pipelines, not an agent
framework. Parallel step execution is automatic when the DAG
allows it (the orchestrator handles fan-out); fan-in is implicit
in the typed signatures. The "after" / "needs" decorators express
explicit ordering when type dependencies don't capture it.

## 6. Telemetry stance

Opt-in. Anonymous usage analytics are enabled by default in the
client (`zenml analytics opt-out` disables; the env var
`ZENML_ANALYTICS_OPT_IN=False` does the same in CI). The local
server stores everything in your own database; ZenML Pro (the
hosted control plane with RBAC + multi-tenancy + Teams + audit
logs) is opt-in and only sees what you push to it. Egress in
self-hosted = your stack components (S3, K8s API, registries) +
whatever ML / LLM SDKs your steps call.

## 7. Token / context strategy

Not applicable at the orchestrator level — ZenML doesn't manage
LLM context, it tracks the artefacts around the LLM call. For
RAG pipelines, the convention is to materialize the retrieved
context as a `Documents` artefact so a downstream eval step can
re-load + re-score the exact context the LLM saw, making LLM
runs reproducible and bisect-able.

## 8. Hot keybinds

None — ZenML is a CLI + dashboard, not a TUI. The dashboard at
`:8237` is a web UI with click-through navigation through
pipelines / runs / artefacts / models. CLI verbs (`zenml
pipeline run`, `zenml stack describe`, `zenml model list`) are
the scriptable surface.

## 9. Killer feature, weakness, when to choose

**Killer feature.** The **stack abstraction**: a single typed
pipeline runs locally on a laptop, on K8s for CI, on Vertex /
SageMaker / Kubeflow for production, by swapping the
orchestrator + artefact store + registry components — pipeline
code never changes. Combined with **automatic artefact
versioning + lineage** across both classical ML and LLM
pipelines, ZenML is one of the few OSS frameworks where "what
data version, model version, prompt version, retrieved-context
version produced this answer in production?" is a click-through
in the dashboard rather than an archaeology project.

**Weakness.** The framework's opinions (typed steps, materialized
artefacts, stack components) are non-trivial to learn — there is
a real onboarding hump versus "just write a script". The LLMOps
surface is younger than the classical-MLOps surface (the prompt /
eval / RAG primitives are still settling), and ZenML Pro's hosted
features (RBAC, Teams, Cloud-managed orchestrators) are how the
project monetizes — the OSS server has the core pipeline +
dashboard but lacks org-grade access control. Stack-component
sprawl (each integration is a separate `pip install`) means
production setups carry a meaningful dependency surface.

**Choose ZenML when** the workload spans both classical ML
training and LLM pipelines and you want **one pipeline framework
+ one dashboard + one lineage graph** covering both, when
"runs in our own cloud, lineage in our own database" is a hard
requirement, or when the team needs to swap orchestrators
(local → K8s → Vertex) without rewriting pipeline code. **Choose
something else when** the workload is purely LLM + agentic and
classical-ML lineage doesn't apply (use
[`langfuse`](../langfuse/) / [`laminar`](../laminar/) for
LLM-only observability), when [`mlflow`](../mlflow/) tracking is
already deeply embedded and you only need experiment tracking
(ZenML overlaps but does more), or when a managed MLOps platform
(Vertex Pipelines / SageMaker Pipelines / Databricks Workflows)
is acceptable and the stack-portability story isn't worth the
framework overhead.
