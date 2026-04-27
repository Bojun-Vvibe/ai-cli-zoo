# prefect

> Python-native **workflow orchestration** as a CLI: turn ordinary Python
> functions into observable, retryable, scheduled flows; serve, deploy,
> and inspect them from the terminal. Pinned to **v3.6.28**
> (commit `e4d25f0bafb3b7499c2acafdc06fc7c5f8c67c9d`,
> [LICENSE](https://github.com/PrefectHQ/prefect/blob/main/LICENSE), Apache-2.0).

Source: <https://github.com/PrefectHQ/prefect>

## TL;DR

`prefect` is the orchestration layer for AI / data pipelines that already
live as Python functions. You decorate your function with `@flow` (and
inner steps with `@task`), then the `prefect` CLI gives you the operator
surface: `prefect server start` boots a local SQLite-backed UI/API at
`127.0.0.1:4200`, `prefect deploy ./my_flow.py:my_flow` registers a
deployment with a schedule (cron / interval / RRULE) and parameters,
`prefect worker start --pool default` polls that deployment and runs it
in a worker pool, and `prefect flow-run logs <id> --tail` streams the
run's logs back to your terminal. Retries, timeouts, caching, and
concurrency limits are decorator kwargs (`@task(retries=3,
retry_delay_seconds=10, cache_key_fn=…)`), not YAML.

## Install

```bash
# pip
pip install -U prefect              # 3.6.28

# uv
uv tool install prefect

# verify
prefect version
```

Python 3.9+. No external DB needed for local use (SQLite by default);
swap to Postgres via `PREFECT_API_DATABASE_CONNECTION_URL` for prod.

## One Concrete Example

`etl.py`:

```python
from prefect import flow, task

@task(retries=3, retry_delay_seconds=5)
def fetch(url: str) -> str:
    import httpx
    return httpx.get(url, timeout=10).text

@task
def count_lines(body: str) -> int:
    return len(body.splitlines())

@flow(log_prints=True)
def etl(url: str = "https://example.com"):
    body = fetch(url)
    n = count_lines(body)
    print(f"{url} → {n} lines")
    return n
```

Operator loop:

```bash
# 1. boot local server + UI
prefect server start &                       # http://127.0.0.1:4200

# 2. one-off run from CLI
python -c "from etl import etl; etl()"

# 3. or deploy on a schedule
prefect deploy etl.py:etl -n hourly --cron "0 * * * *" --pool default
prefect worker start --pool default          # in another terminal

# 4. inspect
prefect deployment ls
prefect flow-run ls --limit 5
prefect flow-run logs <run-id> --tail
```

## Niche It Fills

**The orchestration layer for Python AI pipelines that don't justify
Airflow.** Most LLM / RAG / fine-tuning workflows in this catalog are
"a Python function plus a schedule plus retries plus a UI to see why it
failed last night" — and that is exactly what `prefect` is. The decorator
model means the same `@flow` you call interactively in a notebook is what
the scheduler runs in production; you don't rewrite your code as a DAG.

## Vs Already Cataloged

- **Vs [`dagster`](../dagster/):** `dagster` is asset-centric (you
  declare materialized data products and the graph is derived);
  `prefect` is flow-centric (you write imperative Python and decorate
  it). Pick `dagster` if your team thinks in tables / models; pick
  `prefect` if your team thinks in functions.
- **Vs [`metaflow`](../metaflow/):** both are Python-native, but
  `metaflow` is class-based with `step` methods and is opinionated
  about cards / artifacts for ML experiments; `prefect` is decorator-
  based and aimed at general async orchestration (HTTP, S3, LLM calls)
  more than experiment tracking.
- **Vs [`modal`](../modal/):** `modal` runs your Python on its
  managed cloud; `prefect` orchestrates wherever you already run
  (laptop, ECS, k8s, Modal itself). They compose: a `@flow` whose
  tasks call `modal.Function.lookup(...).remote(...)`.

## Caveats

- The local `prefect server` defaults to SQLite — fine for one operator,
  not for a team. Move to Postgres before sharing the URL.
- Workers are pull-based: a deployment without a running worker in its
  pool will sit in `Late` forever. `prefect worker start --pool <name>`
  is part of the deploy story, not optional infrastructure.
- The free Prefect Cloud tier sends flow-run metadata to Prefect's SaaS
  by default if you `prefect cloud login`. The OSS server (`prefect
  server start`) is fully self-hosted; pick deliberately.
- v2 → v3 was a breaking rewrite (new engine, new agent → worker model,
  removed `Block` deployments). Old tutorials that say `prefect agent
  start` are v2 and will not work against 3.6.x.
