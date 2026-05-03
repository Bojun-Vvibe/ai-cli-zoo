# risingwave

- **Repo:** https://github.com/risingwavelabs/risingwave
- **Version:** v2.8.2 (2026-04-18)
- **License:** Apache-2.0 ([LICENSE](https://github.com/risingwavelabs/risingwave/blob/main/LICENSE))
- **Language:** Rust
- **Install:** prebuilt binary from [GitHub releases](https://github.com/risingwavelabs/risingwave/releases) · `brew install risingwave` · `docker run -it --pull=always -p 4566:4566 -p 5691:5691 risingwavelabs/risingwave:latest single_node` · Helm chart for Kubernetes

## What it does

`risingwave` is a distributed SQL streaming database — Postgres-wire-compatible on the front, Rust-implemented streaming dataflow engine on the back — that lets you create *materialized views* over unbounded streams and have those views maintained incrementally and continuously, instead of recomputed on each query. The mental model: you `CREATE SOURCE` over a streaming input (Kafka, Redpanda, Pulsar, Kinesis, MQTT, MySQL/Postgres CDC, an S3 bucket of files, NATS JetStream, plus more), declare its schema and decoder (JSON, Avro + Schema Registry, Protobuf, CSV, Debezium / Maxwell / Canal envelopes), then write a `CREATE MATERIALIZED VIEW mv AS SELECT ... FROM source GROUP BY ... WINDOW ...` query in standard SQL with stream-aware extensions (tumble / hop / session windows, watermarks, AS OF SYSTEM TIME, temporal joins, interval joins). RisingWave compiles that into a long-running streaming dataflow, persists state to its own storage layer (Hummock — an LSM tree backed by S3 / MinIO / GCS / Azure Blob / local disk), and keeps the materialized view up to date row-by-row as new events arrive. You then either `SELECT * FROM mv` from any Postgres client (`psql`, JDBC, `pg_dump`, BI tools, Grafana via the Postgres datasource) and get sub-second answers from the maintained state, or `CREATE SINK` to push the maintained results out to Kafka / Postgres / MySQL / Iceberg / Delta Lake / Elasticsearch / ClickHouse / Snowflake / Redis / a webhook for downstream consumption. Architecture is a separation-of-compute-and-storage cluster: a meta node, frontend nodes (parse SQL, plan distributed dataflows, terminate Postgres protocol), compute nodes (run the streaming operators), and a compactor (background LSM compaction against the object store). The `risingwave` binary itself ships with a `single_node` mode that runs all roles in one process for dev / small workloads, plus per-role subcommands (`risingwave meta-node`, `risingwave frontend-node`, `risingwave compute-node`, `risingwave compactor-node`) for production deployments. Comes with an embedded SQL workbench at `:5691` and a CLI risectl for cluster admin (rebalance, scale, snapshot meta backup).

## When to pick it / when not to

Pick `risingwave` when you have a stream you want to query in SQL — continuously, incrementally, with results that look like a Postgres table — and the alternative is wiring [Flink](https://flink.apache.org/) jobs in Java/Scala, debugging RocksDB state backends, and running a separate Kafka-Streams + KsqlDB + Postgres sink pipeline. Concrete cases: real-time feature stores for ML inference (windowed aggregates, last-N events per user, joins between a click stream and a user profile table — exposed as a materialized view that an inference service queries with sub-ms latency); operational dashboards (Grafana / Metabase / Superset point at RisingWave as a Postgres datasource and chart live-updating aggregates over the event stream); CDC-driven derived tables (replicate Postgres / MySQL via Debezium → CDC source → materialized join → sink to Iceberg or Postgres for analytics); fraud / anomaly / alerting pipelines that compute "events per minute per account" and emit to a webhook the moment a threshold is crossed; LLM agent telemetry (stream tool-call events into a source, materialize per-agent / per-session aggregates, query for guardrail decisions). Pair with [`benthos`](../benthos/) (stream ETL / event reshaping into Kafka or directly into RisingWave sinks), [`pgweb`](../pgweb/) (browser-based SQL client — RisingWave is Postgres-wire so pgweb works as the ad-hoc query UI), [`duckdb`](../duckdb/) (post-hoc analytical queries against snapshots / Iceberg outputs), and [`atlas`](../atlas/) (manage RisingWave schema-as-code via its Postgres dialect support for the catalog tables).

Skip `risingwave` when your data is bounded, your refresh cadence is "every 5 minutes is fine", and a vanilla Postgres / [`duckdb`](../duckdb/) materialized view (or a cron job re-running a query) is all you need — RisingWave is a streaming engine and the operational surface is real (S3 + 4 node roles, even in small clusters). Skip when your team has zero appetite for object-store-backed state durability tuning (Hummock has knobs — compaction, retention, watermark) and would rather pay for a managed equivalent — RisingWave Cloud and Confluent's KsqlDB / Materialize Cloud are the managed options. Skip when you actually need OLTP — RisingWave is *append-friendly streaming OLAP-MV*, not a transactional database; do not point your application's read-modify-write traffic at it. Skip when you only need windowed aggregates over a single Kafka topic and a Kafka-Streams / `ksqlDB` job is already running and meeting your latency SLO — replacing it with RisingWave for one query is operational churn.

## Vs already cataloged

- **Vs [`duckdb`](../duckdb/):** orthogonal categories. DuckDB is an embedded analytical engine — bounded data, columnar, single process, ideal for ad-hoc analytics on Parquet / CSV / Iceberg snapshots. RisingWave is a distributed streaming MV engine — unbounded data, row-incremental, multi-process. Common pattern: RisingWave maintains the streaming MV → sinks to Iceberg → DuckDB queries the Iceberg table for offline / interactive analytics.
- **Vs [`benthos`](../benthos/):** complementary. Benthos is a single-binary stream ETL / routing / transform tool ("plumbing"). RisingWave is the *stateful* SQL layer on top. Most production setups use Benthos to land / reshape / fan-out events, and RisingWave to compute MVs and joins over them.
- **Vs [`pgweb`](../pgweb/) / [`sqlite-utils`](../sqlite-utils/):** pgweb works directly against RisingWave because RisingWave speaks the Postgres wire protocol — use it as the ad-hoc query UI. sqlite-utils is for SQLite files, not relevant here.
- **Vs [`pocketbase`](../pocketbase/):** different problem space entirely. PocketBase is a single-binary BaaS for app backends; RisingWave is a streaming database for analytical / operational data pipelines.
- **Vs [`bentoml`](../bentoml/) / [`bentoml`-style serving frameworks](../bentoml/):** RisingWave often sits *behind* model-serving — features computed in RisingWave MVs are read by the inference service at request time. The two are not comparable, but they are commonly deployed together.
- **Vs [`pgbackrest`](../pgbackrest/):** different DB. pgbackrest backs up Postgres; RisingWave's durability story is its object-store-backed Hummock state plus meta-node snapshots, configured separately.

## Caveats

- **You must own an object store.** Even single-node mode can use the local disk for Hummock, but any production cluster wants S3 / MinIO / GCS / Azure Blob — that is the durable state layer, not a cache. Plan IAM, lifecycle rules, and bandwidth.
- **Compute / storage separation cuts both ways.** You can scale compute nodes elastically without rebalancing data, which is great. But every read of cold state hits the object store; tune Hummock cache (`storage.block_cache_capacity_mb`) for your working set or tail latencies will surprise you.
- **Postgres-wire ≠ full Postgres.** RisingWave aims at Postgres compatibility for SELECT semantics, but it is not a drop-in for application DBs (no triggers, no row-level updates in the OLTP sense, limited extension surface). Think of it as "Postgres syntax for streaming analytics", not "Postgres".
- **State recovery on crash takes time.** A frontend or compute node restart is fast; a full cluster cold start has to rebuild operator state from Hummock, which can take minutes for large MVs. Run multi-replica meta and compute in production.
- **Watermarks are explicit.** Out-of-order events do not magically heal — declare watermarks on sources and use them in window functions or expect under-counted results at window edges. The docs have a clear chapter; read it before designing windows.
- **Schema evolution on streaming sources is hand-rolled.** When upstream Avro / Protobuf schemas change, you typically `DROP MATERIALIZED VIEW`, `DROP SOURCE`, recreate, and let the new MV bootstrap. Plan for it in CI rather than discovering it at 3 AM.
- Apache-2.0 ([LICENSE](https://github.com/risingwavelabs/risingwave/blob/main/LICENSE)) — permissive; the upstream OSS build is fully Apache-2.0 with no source-available carve-outs (RisingWave Cloud is a separate managed offering).

## Example invocations

```bash
# Single-node dev cluster (frontend on :4566, dashboard on :5691, meta on :5690)
docker run -it --pull=always --name risingwave -p 4566:4566 -p 5691:5691 \
  risingwavelabs/risingwave:latest single_node

# Connect with any Postgres client
psql -h 127.0.0.1 -p 4566 -d dev -U root

# Define a Kafka source (JSON-encoded events)
CREATE SOURCE clicks (
  user_id   VARCHAR,
  url       VARCHAR,
  ts        TIMESTAMPTZ
) WITH (
  connector = 'kafka',
  topic     = 'clicks',
  properties.bootstrap.server = 'redpanda:9092',
  scan.startup.mode = 'earliest'
) FORMAT PLAIN ENCODE JSON;

# Maintain a 1-minute tumbling-window aggregate, incrementally
CREATE MATERIALIZED VIEW clicks_per_minute AS
SELECT
  window_start,
  user_id,
  count(*) AS clicks
FROM TUMBLE(clicks, ts, INTERVAL '1 MINUTE')
GROUP BY window_start, user_id;

# Query the MV like any Postgres table — answers come from maintained state
SELECT * FROM clicks_per_minute
WHERE window_start > now() - INTERVAL '10 MINUTES'
ORDER BY clicks DESC LIMIT 20;

# Sink the MV to a downstream Postgres analytics DB
CREATE SINK clicks_per_minute_sink FROM clicks_per_minute
WITH (
  connector = 'jdbc',
  jdbc.url  = 'jdbc:postgresql://analytics:5432/warehouse?user=rw&password=...',
  table.name = 'clicks_per_minute',
  type = 'upsert',
  primary_key = 'window_start,user_id'
);

# Inspect cluster state via risectl
risectl meta backup-meta            # snapshot meta store
risectl scale resize --include-workers compute --target 6
```

## Why it fits the catalog

RisingWave fills the "streaming SQL database" niche the catalog did not yet cover, distinct from batch analytical engines ([`duckdb`](../duckdb/)), traditional OLTP / BaaS ([`pocketbase`](../pocketbase/)), and stream ETL plumbing ([`benthos`](../benthos/)). For AI / agent telemetry pipelines specifically it is a strong fit — tool-call events, eval metrics, and LLM token-usage streams land in Kafka, RisingWave maintains per-agent / per-session aggregates as MVs, and dashboards or guardrail services query those MVs as if they were Postgres tables. Pair with [`benthos`](../benthos/) for ingest reshaping and [`pgweb`](../pgweb/) for the browser SQL UI.
