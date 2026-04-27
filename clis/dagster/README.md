# dagster

> Snapshot date: 2026-04. Upstream: <https://github.com/dagster-io/dagster>
> Pinned release: `1.13.2` (HEAD `4b5a9b8cfa3a2060a3e44aa163ecd24c38e881fc`).
> License file: [`LICENSE`](https://github.com/dagster-io/dagster/blob/master/LICENSE)
> (sha `82b6b68d6ad4d65757310c2e7dab12ccae01de36`, Apache-2.0).

`dagster` is a Python-shaped orchestration platform whose unit
of code is a typed **asset** — a function that materialises one
named output (a Parquet file, a Postgres table, a fine-tuned
checkpoint, a vector index, an LLM-summarised report) and
declares its upstream assets. The CLI (`dagster`, plus the
embedded `dagster dev` web UI) is the orchestration surface
that catalog RAG / training pipelines reach for once the
prototyping notebook outgrows itself: `dagster asset
materialize`, scheduled sensors, partitioned backfills, and a
typed lineage graph that crosses the LLM / vector-store /
warehouse boundary. It is the catalog's reference for **an
asset-graph orchestrator wired into LLM pipelines** — sibling
to imperative-flow runners like Prefect or Airflow (out of
catalog scope) and a layer above
[`bentoml`](../bentoml/) / [`truss`](../truss/) / [`vllm`](../vllm/)
on the serving side.

## 1. Install footprint

- `pip install dagster dagster-webserver` (Python 3.9–3.12)
  installs the `dagster` CLI plus the `dagster-webserver` UI on
  `127.0.0.1:3000`.
- `dagster dev -f my_assets.py` boots a local instance with the
  webserver, daemon, and SQLite-backed run / event log.
- LLM-adjacent extras are separately versioned packages:
  `dagster-openai`, `dagster-anthropic`, `dagster-langchain`,
  `dagster-llama-index`, `dagster-pinecone`, `dagster-weaviate`,
  `dagster-snowflake`, `dagster-dbt`, `dagster-duckdb`,
  `dagster-embedded-elt`, etc. (See the `dagster-io/dagster`
  `python_modules/libraries/` tree for the full list pinned at
  this snapshot.)
- Production deploys land on Kubernetes (`dagster-k8s`), ECS
  (`dagster-aws`), or Dagster+ (the hosted control plane).

## 2. Repo, version, license

- Repo: <https://github.com/dagster-io/dagster>
- Latest release: `1.13.2` (2026-04-23).
- HEAD pinned at this snapshot:
  `4b5a9b8cfa3a2060a3e44aa163ecd24c38e881fc`.
- License: Apache-2.0. Pinned LICENSE blob:
  [`LICENSE`](https://github.com/dagster-io/dagster/blob/master/LICENSE)
  (sha `82b6b68d6ad4d65757310c2e7dab12ccae01de36`).

## 3. What it actually does

Verbs are organised around the asset graph and the run that
materialises it:

```
dagster dev                          # boot webserver + daemon locally
dagster asset materialize --select   # materialise one or more assets
dagster job execute -j ingest_job    # run a legacy op-based job
dagster sensor list / start / stop   # event-driven triggers
dagster schedule list / start / stop # cron-shaped triggers
dagster run list / wipe              # browse / clean run history
dagster definitions validate         # static-check the asset graph
dagster project scaffold             # new repo skeleton
dagster-daemon run                   # the supervisor process
```

A typical LLM pipeline expressed as assets: `raw_pdfs`
(materialised by an S3 sensor) → `cleaned_text` (a Python
function calling a `marker` / `unstructured` parser) →
`embedded_chunks` (a `dagster-openai` resource emits embeddings,
written to Pinecone via the `dagster-pinecone` resource) →
`eval_report` (a downstream asset that runs a
[`deepeval`](../deepeval/) suite against a held-out set and
fails the run if scores regress). The lineage between those
five names is what `dagster` actually owns — re-running
`eval_report` automatically rewinds to the cheapest upstream
that changed.

## 4. MCP support

None first-party. `dagster` is an orchestrator, not an agent
runtime; an LLM agent that wants to drive `dagster` does so by
calling the GraphQL API (`dagster-graphql` package) or the
REST sensor endpoints. Community MCP wrappers exist as small
adapters but are not part of the upstream.

## 5. Sub-agent model

None at the orchestrator level. The relevant primitive is
**partitioned backfills + dynamic asset graphs**: a single
asset can fan out to N partitions (one per S3 prefix, one per
day, one per tenant), each materialised in parallel by the
daemon, with retries and run-level concurrency limits enforced
by `QueuedRunCoordinator`. For LLM pipelines that means "embed
3000 PDFs in parallel with a 20-concurrent cap and per-failure
retry" is a config knob, not an orchestrator-loop you write.

## 6. Telemetry stance

On by default, opt-out. The OSS instance posts anonymous usage
events to `telemetry.elementl.com` on `dagster dev` startup;
flip off with `telemetry: enabled: false` in `dagster.yaml` or
`DAGSTER_DISABLE_TELEMETRY=1`. The hosted Dagster+ control
plane is opt-in via `dagster-cloud` agent. Run / event logs
land in the configured storage (SQLite locally, Postgres in
prod) and never leave the deployment.

## 7. Token / context strategy

N/A at the orchestrator. LLM context windows live in the
asset's compute function — `dagster-openai` / `dagster-anthropic`
resources are thin clients, the function decides chunking,
caching, and retries. The orchestration win is **memoization
via versioning**: tag an asset with a `code_version`, and
`dagster` will refuse to re-materialise a downstream whose
upstream code+data versions are unchanged, so a stable
embedding pipeline does not re-embed 3000 PDFs every nightly
run.

## 8. Hot keybinds

None — `dagster` is a non-interactive command-runner. The
`dagster-webserver` UI (`dagster dev`) is the clickable
surface; the CLI is for CI and headless ops.

## 9. Killer feature, weakness, when to choose

**Killer feature.** **The asset graph as the source of truth,
with software-defined-asset semantics that include lineage,
partitions, code versions, and run-level retries** — the same
graph that materialises `eval_report` knows that `raw_pdfs`
upstream changed, and re-runs only the affected sub-graph.
Most LLM pipelines start as "a notebook calling OpenAI in a
loop" and grow into "a notebook calling OpenAI in a loop on a
cron, with no idea which embeddings are stale"; `dagster`
absorbs that growth without rewriting the pipeline as a DAG of
opaque tasks (the Airflow / Prefect-imperative path). Typed
asset I/O managers + partitions + backfills are the load-
bearing primitives the rest of the catalog assumes someone
else owns.

**Weakness.** It is **a heavy ceremony for a 200-line script**
— the asset / resource / definitions / IO-manager mental model
is correct for a 50-asset RAG pipeline but overkill for a
prototype. The webserver + daemon + Postgres footprint is
substantial; the hosted Dagster+ tier is where the polish
actually lives (branch deployments, alert policies, run insights),
and several niceties bleed into "Dagster+ only".

**Choose `dagster` when** the LLM pipeline has more than one
asset, more than one schedule, or any partitioned backfill
shape — and the team values typed lineage and code-versioned
memoisation over imperative flow control. **Choose something
else when** the pipeline is one cron job and one Python file
(plain `cron` + a script, or Prefect / Airflow), the
workload is single-task agent loops ([`burr`](../burr/),
[`langgraph`](../langgraph/)), or the orchestration is really
"deploy and serve" not "schedule and materialise" (
[`truss`](../truss/), [`bentoml`](../bentoml/),
[`vllm`](../vllm/)).
