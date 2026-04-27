# kedro

> Snapshot date: 2026-04. Upstream: <https://github.com/kedro-org/kedro>
> Pinned release: `1.3.1` (HEAD `c3d3dea18837d2cca6a55a9fd079e61e0bbb5da1`).
> License file: [`LICENSE.md`](https://github.com/kedro-org/kedro/blob/main/LICENSE.md)
> (Apache-2.0).

`kedro` is a Python framework + CLI from the LF AI & Data
Foundation that imposes an opinionated **project template + data
catalog + node/pipeline abstraction** on data-science codebases
so notebooks graduate into reproducible, testable pipelines
without bolting on bespoke glue. The CLI (`kedro new`,
`kedro run`, `kedro viz`, `kedro catalog`) is the entry point
for scaffolding projects, materialising the typed dataset
catalog, executing pipeline DAGs against pluggable runners
(sequential / parallel / Spark / Airflow / Argo / Dask), and
exporting interactive lineage visualisations. It is the catalog's
reference for **a notebook-to-pipeline framework with a
declarative data catalog and pluggable runners** — sibling to
asset-graph orchestrators like [`dagster`](../dagster/) and
typed pipeline runners like [`zenml`](../zenml/), with the
narrower scope of "structure my analytics codebase" rather than
"own the production scheduler".

## 1. Install footprint

- `pip install kedro` (Python 3.9–3.13). Adds `kedro-datasets`
  out-of-band when you want first-party connector classes
  (`pip install "kedro-datasets[pandas.CSVDataset,spark.SparkDataset]"`).
- `kedro new --starter=spaceflights-pandas` scaffolds a runnable
  example project; `kedro viz run` boots a local web UI on
  `:4141` rendering the typed pipeline + dataset graph.
- Plugins: `kedro-airflow`, `kedro-docker`, `kedro-mlflow`,
  `kedro-databricks`, `kedro-azureml`, `kedro-vertexai`,
  `kedro-sagemaker` translate a Kedro project into the target
  platform's native job spec.
- LLM-adjacent use: a typical RAG pipeline expressed as Kedro
  nodes (`load_documents` → `chunk` → `embed` → `index` →
  `evaluate`) with the embedding model + vector store + eval
  metrics swappable via the YAML `catalog.yml` and `parameters.yml`.

## 2. Repo, version, license

- Repo: <https://github.com/kedro-org/kedro>
- Latest release: `1.3.1` (2026-04-07).
- HEAD pinned at this snapshot:
  `c3d3dea18837d2cca6a55a9fd079e61e0bbb5da1`.
- License: Apache-2.0. License file at
  [`LICENSE.md`](https://github.com/kedro-org/kedro/blob/main/LICENSE.md).
- Governance: LF AI & Data Foundation (graduated project).

## 3. What it actually does

The CLI verbs map to project-shaped operations:

```
kedro new --starter=<name>          # scaffold a project from a template
kedro run                           # execute the default pipeline
kedro run --pipeline=ingest         # run a named sub-pipeline
kedro run --from-nodes=embed        # partial re-run from a node
kedro catalog list / resolve        # inspect the typed dataset catalog
kedro pipeline create <name>        # scaffold a modular pipeline
kedro registry list                 # list registered pipelines
kedro viz run                       # local lineage UI on :4141
kedro ipython / kedro jupyter       # session preloaded with catalog
kedro test / kedro lint             # opinionated quality gates
```

A `node` is a typed Python function. A `pipeline` is a
collection of nodes wired by named dataset inputs/outputs. A
`DataCatalog` (declared in `conf/base/catalog.yml`) maps those
names to concrete I/O classes (`pandas.CSVDataset`,
`spark.SparkDataset`, `pickle.PickleDataset`,
`partitions.PartitionedDataset`, custom `AbstractDataset`
subclasses) so the same node code reads from local files in
dev, S3 in staging, and a warehouse table in prod with zero
code changes. `parameters.yml` is the equivalent for
hyperparameters / model names / prompt templates.

## 4. MCP support

None first-party. Kedro sits at the pipeline-structure layer; if
a node calls an MCP-aware agent ([`pydantic-ai`](../pydantic-ai/),
[`fast-agent`](../fast-agent/)) the MCP traffic happens inside
the node and Kedro tracks only the typed inputs/outputs.

## 5. Sub-agent model

None — Kedro is a pipeline framework, not an agent runtime.
Parallel node execution is handled by the `ParallelRunner` /
`ThreadRunner` against a topologically-sorted DAG; fan-in is
implicit in the typed signatures.

## 6. Telemetry stance

Opt-in via the `kedro-telemetry` plugin (installed as a default
dependency since 0.18). On first `kedro` invocation the CLI
prompts y/N; declining writes `consent: false` to
`.telemetry` and no events are sent. The `DO_NOT_TRACK=1`
env var is also honoured. Events captured (when consented):
CLI command name + project hash, never dataset content,
parameter values, or code.

## 7. When it is the right answer

- You have a Jupyter notebook that has outgrown itself and you
  want a typed catalog + pipeline structure without rewriting
  to [`dagster`](../dagster/) / [`zenml`](../zenml/) /
  [`prefect`/Airflow](https://airflow.apache.org/).
- You want the same pipeline code to run locally on pandas,
  on a cluster on Spark, and on Airflow / Argo / Vertex /
  SageMaker / Databricks via a plugin — without an opinionated
  SaaS control plane.
- The team values "this looks like a normal Python project with
  conventions" over "this is a new orchestration product".

## 8. When to reach for something else

- You need an event-driven sensor / scheduler / partitioned
  backfill UI as part of the framework — that is
  [`dagster`](../dagster/)'s lane.
- You want LLMOps-shaped artefact + prompt + eval-run tracking
  in the same dashboard as classical training runs —
  [`zenml`](../zenml/) is closer.
- You only need a single-purpose ML lifecycle tracker (params /
  metrics / artefacts / model registry) without the project
  template — [`mlflow`](../mlflow/) is the lighter answer.

## 9. Cross-references

- [`dagster`](../dagster/) — asset-graph orchestrator with a
  richer scheduling + sensor surface.
- [`zenml`](../zenml/) — pipeline-as-code MLOps with a swappable
  execution stack and unified ML+LLM dashboard.
- [`mlflow`](../mlflow/) — narrower experiment-tracking + model
  registry without the project template.
- [`metaflow`](../metaflow/) — Netflix's flow-shaped sibling
  with a stronger HPC + GPU bias.
