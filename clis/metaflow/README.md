# metaflow

> Snapshot date: 2026-04. Upstream: <https://github.com/Netflix/metaflow>
> Pinned release: `2.19.22` (HEAD `34f2de8b02db671aa0ab5e98f55ce89409229d41`).
> License file: [`LICENSE`](https://github.com/Netflix/metaflow/blob/master/LICENSE)
> (Apache-2.0).

`metaflow` is the Python framework + CLI Netflix open-sourced in
2019 for building and running data-science / ML / LLM workflows
that scale from a laptop loop to thousands of GPUs without
rewriting the user code. The unit of code is a `FlowSpec`
subclass whose methods are decorated `@step`s wired by `self.next(...)`
calls; the `python myflow.py run` CLI dispatches that DAG to a
local subprocess pool, AWS Batch, Kubernetes, Argo Workflows,
Step Functions, or Airflow with the same source. It is the
catalog's reference for **a flow-shaped, decorator-driven ML
framework with first-class GPU + HPC dispatch and built-in
artifact / namespace versioning** — sibling to asset-graph
orchestrators like [`dagster`](../dagster/) and project-shaped
frameworks like [`kedro`](../kedro/), with a stronger bias
toward "I want to spray this training job across 1,000 GPUs on
AWS Batch and resume from any step".

## 1. Install footprint

- `pip install metaflow` (Python 3.8–3.13). The CLI is
  per-flow: `python myflow.py run`, `python myflow.py resume`,
  `python myflow.py logs`. A separate `metaflow` CLI exposes
  config + tutorials (`metaflow tutorials pull`,
  `metaflow configure aws`).
- Local-only mode is zero-config: artifacts land in
  `.metaflow/` as pickled blobs, run metadata in a SQLite
  database; the local UI is `metaflow-ui` (separate package)
  on `:3000`.
- Cloud mode requires a stack: artifact store (S3 / Azure Blob
  / GCS), metadata service (REST API + Postgres), and a
  compute backend (`@batch`, `@kubernetes`, `@nvidia` for
  Nebius / CoreWeave-style GPU pools, `@argo_workflows`,
  `@step_functions`).
- Plugins shipped in-tree: `@card` for HTML reports per step,
  `@checkpoint` for fault-tolerant long-running jobs,
  `@huggingface_hub` for model weight caching,
  `@pypi` / `@conda` for hermetic per-step environments.

## 2. Repo, version, license

- Repo: <https://github.com/Netflix/metaflow>
- Latest release: `2.19.22` (2026-03-20).
- HEAD pinned at this snapshot:
  `34f2de8b02db671aa0ab5e98f55ce89409229d41`.
- License: Apache-2.0. License file at
  [`LICENSE`](https://github.com/Netflix/metaflow/blob/master/LICENSE).
- Maintained by Netflix's ML platform team + Outerbounds (the
  hosted commercial offering); the OSS package is fully usable
  standalone.

## 3. What it actually does

A flow is a Python class:

```python
from metaflow import FlowSpec, step, batch, retry, card

class TrainFlow(FlowSpec):
    @step
    def start(self):
        self.shards = list(range(8))
        self.next(self.train, foreach='shards')

    @batch(gpu=1, memory=60000, image='pytorch/pytorch:2.5.1-cuda12.4')
    @retry(times=2)
    @step
    def train(self):
        # self.input is the shard id; train one shard
        self.metric = ...
        self.next(self.join)

    @step
    def join(self, inputs):
        self.metrics = [i.metric for i in inputs]
        self.next(self.end)

    @card
    @step
    def end(self):
        pass

if __name__ == '__main__':
    TrainFlow()
```

`python trainflow.py run` fans out 8 GPU containers on AWS
Batch, retries flaky steps, joins the results, renders an HTML
report, and writes every step's `self.*` attributes as
versioned artifacts you can later open as
`Run('TrainFlow/123').data.metrics`. `resume` re-executes from
the failed step using cached upstream artifacts. The 2024+
LLM-shaped use is identical: replace `train` with a
fine-tuning step calling `transformers` / `unsloth` / `trl`,
or a batch-eval step calling [`deepeval`](../deepeval/) /
[`promptfoo`](../promptfoo/) on a held-out set.

## 4. MCP support

None first-party. Metaflow owns the workflow + dispatch layer;
if a `@step` invokes an MCP-aware agent
([`pydantic-ai`](../pydantic-ai/), [`fast-agent`](../fast-agent/))
the MCP traffic happens inside the step container.

## 5. Sub-agent model

None — Metaflow is a flow runner, not an agent framework.
Parallelism is the `foreach` / `join` pattern over a static
fan-out width determined at runtime; nested foreach is
supported. No long-lived sub-processes inside a step beyond
what the user code spawns.

## 6. Telemetry stance

Opt-out, off-by-default for OSS. The standalone OSS package
ships with no telemetry collection. The hosted Outerbounds
offering and a self-hosted metadata service obviously see your
flow names + run IDs + artifact metadata (not artifact
content), but that is the metadata service you yourself
deployed. The `METAFLOW_TELEMETRY=...` envvar is reserved for
plugin-shipped collectors and unused by mainline.

## 7. When it is the right answer

- You have GPU-heavy training / fine-tuning / batch-inference
  jobs that need to run hundreds-to-thousands wide on
  AWS Batch / Kubernetes / Argo, and you want the same code to
  run locally for dev with `--datastore=local`.
- You want **artifact-by-default versioning** — every `self.x`
  in every step is a queryable, time-travel-able artifact,
  for free, without an `mlflow.log_artifact()` call.
- You value `python myflow.py resume` (resume from the failed
  step with cached upstreams) as a first-class verb.

## 8. When to reach for something else

- You need an event-driven sensor / asset-graph + partitioned
  backfill UI — [`dagster`](../dagster/) is closer.
- You want a project template + typed data catalog as the
  organising abstraction rather than `FlowSpec` subclasses —
  [`kedro`](../kedro/) is the better fit.
- You only need experiment / model-registry tracking without
  a workflow framework — [`mlflow`](../mlflow/).
- You want the workflow framework to also own LLM-side
  prompt / eval-run versioning in the same dashboard as ML
  runs — [`zenml`](../zenml/).

## 9. Cross-references

- [`dagster`](../dagster/) — asset-graph orchestrator with a
  richer scheduling / sensor surface.
- [`kedro`](../kedro/) — project-template + typed data catalog
  framework, lighter on dispatch.
- [`zenml`](../zenml/) — pipeline-as-code MLOps with unified
  ML+LLM dashboard.
- [`mlflow`](../mlflow/) — narrower experiment-tracking +
  model registry.
- [`skypilot`](../skypilot/) — multi-cloud GPU job dispatcher
  Metaflow can compose with for the bursty batch lane.
