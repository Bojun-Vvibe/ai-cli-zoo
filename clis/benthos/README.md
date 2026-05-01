# benthos

> **Stream processor where the whole pipeline is one YAML file:
> N inputs → declarative processors → N outputs, with backpressure
> and at-least-once delivery wired in by the engine** — a single
> Go binary (originally `Jeffail/benthos`, continued under
> Redpanda as `redpanda-data/connect`, with the upstream
> `warpstreamlabs/bento` Apache-2.0 fork preserving the open
> ecosystem). Pinned to **bento v1.10.0** (released 2025-08-12,
> [LICENSE](https://github.com/warpstreamlabs/bento/blob/main/licenses/rcl.md),
> Apache-2.0 for core; some enterprise components separately
> licensed in the Redpanda fork).

Source: <https://github.com/warpstreamlabs/bento>
(Redpanda fork: <https://github.com/redpanda-data/connect>)

## TL;DR

`bento` (the OSS continuation of `benthos`) is the answer when
you have data arriving on Kafka / NATS / SQS / Redis / Pulsar /
MQTT / a webhook / a Postgres CDC stream and you need to reshape
it, enrich it, branch it, batch it, and write it to a different
set of sinks — without writing a Java Flink topology or a Python
worker fleet. One YAML file declares the inputs, the processor
chain (filter / map / `bloblang` transforms / HTTP enrich /
`branch` parallel side-effects / `workflow` DAG), and the outputs;
the engine handles batching, backpressure, retries with
exponential backoff, dead-letter-queue routing, distributed tracing,
and Prometheus metrics out of the box. Stateless by design — the
state lives in the brokers and stores you wire it to — so a
binary restart is free and horizontal scale is "run more pods".

## Install

```bash
# Homebrew (macOS / Linux) — installs the bento OSS fork
brew install bento

# Static binary (any OS)
# https://github.com/warpstreamlabs/bento/releases
# Redpanda Connect fork: https://github.com/redpanda-data/connect/releases

# Docker
docker pull ghcr.io/warpstreamlabs/bento:latest

# verify
bento --version    # bento version 1.10.0
```

## Examples

```bash
# Kafka topic → JSON transform → Postgres + S3 in one config
cat > pipeline.yaml <<'YAML'
input:
  kafka:
    addresses: [ broker:9092 ]
    topics: [ events ]
    consumer_group: bento-rollup

pipeline:
  processors:
    - bloblang: |
        root.event_id = this.id
        root.user = this.payload.user.id
        root.ts = this.timestamp.ts_parse("2006-01-02T15:04:05Z")
        root.shard = this.user.hash("xxhash64") % 16
    - bloblang: 'meta s3_key = "events/%s/%s.json".format(this.shard, this.event_id)'

output:
  broker:
    pattern: fan_out
    outputs:
      - sql_insert:
          driver: postgres
          dsn: ${POSTGRES_DSN}
          table: events
          columns: [ event_id, user, ts ]
          args_mapping: 'root = [ this.event_id, this.user, this.ts ]'
      - aws_s3:
          bucket: events-archive
          path: ${! meta("s3_key") }

metrics: { prometheus: {} }
tracer:  { open_telemetry_collector: { url: otel:4317 } }
YAML

bento -c pipeline.yaml

# lint a config without running it
bento lint pipeline.yaml

# unit-test a bloblang transform with declarative cases
bento test ./tests/

# generate a starter config from input/output names
bento create kafka/jq/sql_insert > skeleton.yaml
```

## Use when

- You need a **Kafka → enrich → Postgres** (or Kafka → S3, NATS
  → Elasticsearch, MQTT → Kafka, Postgres CDC → BigQuery, SQS
  → webhook fan-out) pipeline and the team does not have the
  Flink / Spark / Beam operational appetite — bento is one Go
  binary, one YAML file, restart-safe.
- The transformation surface is **field renames + type coercion
  + JSON reshaping + lookups + branching** — bloblang is purpose-
  built for this and an order of magnitude faster to write than
  jq + a wrapping script.
- You want **at-least-once delivery, automatic batching, and DLQ
  routing as engine features** rather than concerns you reinvent
  per-pipeline — bento ships them on every input/output.
- You need to **fan one event into N side effects in parallel**
  (write to DB, push to webhook, archive to S3, emit metric) and
  have the failure of one branch not block the others —
  `output: broker: pattern: fan_out` covers it.
- Pair with [`vector`](../vector/) (vector excels at logs +
  metrics observability piping; bento excels at event-shaped JSON
  business pipelines) and [`kcat`](../kcat/) (kafkacat / kcat to
  poke topics by hand while iterating on a bento config).

Skip `bento` when the workload is **stateful stream joins /
windowed aggregations across millions of keys** (use Flink /
Materialize / RisingWave), when you only need a one-shot batch
ETL (use [`duckdb`](../duckdb/) / [`dlt`](../dlt/) / a Python
script), or when a single Lambda / Cloud Run handler is the right
size of solution.
