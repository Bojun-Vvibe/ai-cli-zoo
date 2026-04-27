# dlt

> Snapshot date: 2026-04. Upstream: <https://github.com/dlt-hub/dlt>
> Pinned release: `1.25.0` (HEAD `488bd3946fe912d43185b98e28638bc6d89cab80`).
> License file: [`LICENSE.txt`](https://github.com/dlt-hub/dlt/blob/devel/LICENSE.txt)
> (Apache-2.0).

`dlt` (data load tool) is an Apache-2.0 Python library + CLI from
dlt-hub for declarative, schema-evolving extract-and-load pipelines.
You write a Python generator that yields rows or nested dicts
(`@dlt.resource`), point it at a destination (`duckdb`, `postgres`,
`bigquery`, `snowflake`, `redshift`, `databricks`, `motherduck`,
`clickhouse`, `synapse`, `athena`, `qdrant`, `weaviate`, `lancedb`,
`filesystem` for parquet/JSONL on S3/GCS/Azure), and `pipeline.run(...)`
infers the schema, normalises nested structures into typed child
tables, handles incremental state, and ships the rows in a single
transactional load. The catalog's reference for **"I have an API /
SaaS / database I want to land into a warehouse or vector store
without authoring DDL or a Singer tap"**, sibling to the
heavyweight platforms ([Airbyte](https://airbyte.com/),
[Meltano](https://meltano.com/), [Fivetran](https://fivetran.com/))
that own the same problem at a different shape — `dlt` is a
library you `import`, not a daemon you operate.

## 1. Install footprint

- `pip install dlt` (Python 3.9–3.13). Adds destination extras on
  demand: `pip install "dlt[duckdb,postgres,parquet,filesystem,s3]"`
  pulls in only the wheels you need; the core install stays under
  20 MB.
- `dlt init <source> <destination>` scaffolds a runnable pipeline
  (`dlt init github duckdb` writes a `github_pipeline.py` against
  the GitHub REST source and a local `duckdb` warehouse).
- `dlt deploy <pipeline.py> github-action --schedule "*/30 * * * *"`
  generates a runnable `.github/workflows/` file that boots the
  pipeline on schedule with secrets wired through repo settings.
- LLM-adjacent shapes: native `lancedb` / `qdrant` / `weaviate`
  destinations turn a `@dlt.resource` into a vector store load;
  pair with [`text-embeddings-inference`](../text-embeddings-inference/)
  upstream for the embedding step and `dlt` handles the
  schema + idempotent merge into the vector index.

## 2. Repo, version, license

- Repo: <https://github.com/dlt-hub/dlt>
- Latest release: `1.25.0` (2026-04-15).
- HEAD pinned at this snapshot:
  `488bd3946fe912d43185b98e28638bc6d89cab80`.
- License: Apache-2.0. License file at
  [`LICENSE.txt`](https://github.com/dlt-hub/dlt/blob/devel/LICENSE.txt).
- Default branch: `devel` (releases cut from `master`).

## 3. What it actually does

```python
import dlt
from dlt.sources.helpers import requests

@dlt.resource(write_disposition="merge", primary_key="id")
def issues(repo: str, since=dlt.sources.incremental("updated_at")):
    url = f"https://api.github.com/repos/{repo}/issues"
    while url:
        r = requests.get(url, params={"since": since.last_value,
                                      "state": "all"})
        yield r.json()
        url = r.links.get("next", {}).get("url")

pipeline = dlt.pipeline(
    pipeline_name="gh_issues",
    destination="duckdb",
    dataset_name="github_raw",
)
load_info = pipeline.run(issues("dlt-hub/dlt"))
print(load_info)
```

That eight-line script: paginates the GitHub REST API, materialises
nested label/assignee arrays into typed child tables
(`github_raw.issues`, `github_raw.issues__labels`,
`github_raw.issues__assignees`) with foreign keys, persists the
`updated_at` watermark in a `_dlt_pipeline_state` table, performs
a transactional merge on `id`, and lands the result in a local
`gh_issues.duckdb` file. Re-run picks up only rows newer than
the watermark.

## 4. MCP support

None first-party. `dlt` is the load tier; an LLM agent that
calls `pipeline.run(...)` does so as a normal Python tool. The
schema-evolution + state-tracking story is the value, not the
agent loop.

## 5. Sub-agent model

None — pipeline runtime, not an agent. The relevant primitive is
**parallel resource extraction + parallel load workers** controlled
by `RuntimeConfiguration` (`workers`, `max_parallel_items`); a
single `pipeline.run(...)` fans out across resources in a source
and loads in concurrent jobs per destination.

## 6. Telemetry stance

Anonymous usage telemetry on by default in OSS (Sentry-shaped
runtime errors + counts of pipeline runs / destinations / sources;
no row content, no schemas, no credentials). Opt-out per project
via `config.toml`:

```toml
[runtime]
dlthub_telemetry = false
```

or globally with `DLT_TELEMETRY=false`. Hosted dlt+ control plane
is opt-in via separate signup.

## 7. When it is the right answer

- You want to land a third-party API or production database into
  a warehouse / lakehouse / vector store and would rather write
  20 lines of Python than configure a Singer tap or stand up
  Airbyte.
- You need typed schema evolution (new field appears upstream → new
  column appears downstream, with no manual migration) plus
  transactional incremental merge as table stakes.
- The same pipeline needs to run locally (`destination="duckdb"`),
  in CI (`destination="filesystem"` writing parquet to S3), and
  in prod (`destination="snowflake"`) by changing one argument.

## 8. When to reach for something else

- You want a hosted UI with non-engineer-friendly connector
  catalogue and per-row pricing — that's [Fivetran](https://fivetran.com/)
  or [Airbyte Cloud](https://airbyte.com/).
- You need a streaming CDC pipeline with sub-second latency —
  [Debezium](https://debezium.io/) + Kafka is the right shape;
  `dlt` is batch-oriented (minute / hour cadence).
- Your transformations dominate and the load is trivial —
  [`dbt`](https://www.getdbt.com/) owns the T; `dlt` is the EL
  before it.

## 9. Cross-references

- [`dagster`](../dagster/) — orchestrator with first-class
  `dagster-embedded-elt` integration for `dlt` pipelines as
  software-defined assets.
- [`text-embeddings-inference`](../text-embeddings-inference/) —
  high-throughput embedding server `dlt`'s `lancedb` / `qdrant`
  / `weaviate` destinations call upstream.
- [`duckdb`](https://duckdb.org/) — the default zero-config
  destination for `dlt init` scaffolds; the local-loop equivalent
  of "real warehouse" semantics.
