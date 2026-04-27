# rill

> **Operational BI as code**: a single Go binary that takes Parquet /
> CSV / DuckDB / ClickHouse / Druid / Pinot / BigQuery / Snowflake /
> Postgres sources, declarative YAML dashboards, and serves a
> sub-second OLAP UI on `localhost:9009` — no warehouse, no JS build,
> no separate semantic layer. Pinned to **v0.85.3**
> (commit `bb2945c5abdc84da3b86aab8970691d2db2714ae`,
> [LICENSE.md](https://github.com/rilldata/rill/blob/main/LICENSE.md), Apache-2.0).

Source: <https://github.com/rilldata/rill>

## TL;DR

`rill` is "metabase / superset / lookml as one local binary":
`rill start my-project` reads `sources/*.yaml`, `models/*.sql`, and
`dashboards/*.yaml`, wires them through a local DuckDB engine
(default) or a remote OLAP target, and boots a fast operational
dashboard at `127.0.0.1:9009`. Definitions are *files in git*: a
**source** is a YAML connector spec (`type: s3`, `path: s3://…/*.parquet`),
a **model** is a `.sql` file (DuckDB or ClickHouse SQL — Rill picks the
dialect from your `connector`), and a **dashboard** (technically a
"metrics view" + a dashboard YAML) declares dimensions, measures,
time grain, and default filters. Edits to any of those files
hot-reload the running UI. `rill deploy` ships the same project
to managed Rill Cloud where the dashboard is shareable; the OSS
binary works fully offline.

## Install

```bash
# direct (single binary)
curl -fsSL https://rill.sh | sh     # 0.85.3

# brew
brew install rilldata/tap/rill

# verify
rill --version
```

No runtime dependencies. The bundled DuckDB makes "I have a Parquet
file" → "I have a dashboard" a one-binary path.

## One Concrete Example

```bash
rill start my-dash --create
cd my-dash
```

`sources/events.yaml`:

```yaml
type: local_file
path: data/events.parquet
```

`models/events_daily.sql`:

```sql
select
  date_trunc('day', event_ts) as day,
  country,
  status,
  count(*) as n,
  sum(latency_ms) as total_latency_ms
from events
group by 1, 2, 3
```

`dashboards/events_daily.yaml`:

```yaml
type: metrics_view
model: events_daily
timeseries: day
dimensions:
  - column: country
  - column: status
measures:
  - name: requests
    expression: sum(n)
    format_preset: humanize
  - name: avg_latency
    expression: sum(total_latency_ms) / sum(n)
    format_d3: ".2f"
```

```bash
rill start              # http://127.0.0.1:9009
```

Navigate to `events_daily` — time-series chart, dimension
breakdowns, leaderboards, time-comparison, all derived from the
metrics view. Edit `events_daily.sql` in your editor → reload.

## Niche It Fills

**The "give me a real dashboard over my Parquet / CSV files in five
minutes, in git" tool.** AI / data pipelines in this catalog
([`dlt`](../dlt/), [`prefect`](../prefect/), [`dagster`](../dagster/),
[`metaflow`](../metaflow/)) land their output as Parquet / DuckDB /
warehouse tables. The downstream question — "what does the data
look like, sliced by these three dimensions, over time" — usually
gets answered by spinning up Superset / Metabase (heavy server, JS
build, separate auth) or by writing a one-off notebook. `rill`
fills the gap: declarative YAML dashboards in the same repo as the
SQL that produced them, version-controlled, code-reviewable, and
runnable from one binary.

## Vs Already Cataloged

- **Vs [`harlequin`](../harlequin/):** `harlequin` is a SQL editor
  for ad-hoc exploration; `rill` produces persistent dashboards
  derived from declarative YAML. Use `harlequin` to prototype
  the SQL, then promote it into a `rill` model.
- **Vs [`datasette`](../datasette/):** `datasette` exposes a
  SQLite file as a JSON API + browseable tables (transactional /
  document-shaped data); `rill` builds OLAP dashboards
  (time-series, dimension drill-downs, measures) over column-store
  data. Different shapes of "publish data on a port".
- **Vs [`evidently`](../evidently/) / [`trulens`](../trulens/):**
  those are model / LLM observability dashboards with built-in
  metrics; `rill` is generic BI where you author the metrics.
  For ML metrics use the specialized ones; for product / business
  / pipeline-output metrics use `rill`.
- **Vs Superset / Metabase:** `rill` is single-binary, code-first,
  Parquet/DuckDB-native, sub-second on a laptop. Superset /
  Metabase are server-first, UI-first, multi-user with
  permissions. Pick `rill` for the "operator + analyst" workflow,
  pick Superset / Metabase when "non-technical people log in to
  edit dashboards" is the requirement.

## Caveats

- The OSS binary stores no auth — the local UI is bind-on-localhost
  by default. Exposing it requires a reverse proxy with auth, or
  promoting to managed Rill Cloud (the SaaS tier).
- Anonymous telemetry is **on by default**; opt out with
  `rill telemetry disable` (writes to `~/.rill/config.yaml`) or
  `RILL_TELEMETRY=0`. The OSS code path that emits it is in the
  repo and verifiable.
- Metrics views inherit the SQL dialect of the underlying connector:
  a model on a DuckDB source uses DuckDB SQL, on ClickHouse uses
  ClickHouse SQL. Mixing connectors in one project means mixing
  dialects in adjacent files — keep models per-connector.
- Hot-reload watches the project tree; if you store the project
  on a network mount with weak FS event support (some NFS / SMB
  setups), edits won't propagate without restarting `rill start`.
