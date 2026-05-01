# kcat

## What it does
A non-JVM, single-binary producer/consumer/metadata client for Apache Kafka (and any Kafka-protocol-compatible broker: Redpanda, WarpStream, Confluent Cloud, MSK). `kcat -P -b broker -t topic` streams stdin into a topic; `kcat -C -b broker -t topic -o beginning` consumes from earliest offset; `kcat -L -b broker` dumps cluster metadata (brokers, partitions, ISRs, leaders) as JSON. Speaks plaintext, SASL/PLAIN, SASL/SCRAM, SSL, and OAUTHBEARER, and ships with built-in Avro / JSON Schema / Protobuf deserialization when linked against `libserdes`.

## Why it's interesting
Where the official `kafka-console-producer.sh` / `kafka-console-consumer.sh` shell scripts boot a full JVM per invocation (multi-second cold start, 200+ MB resident), `kcat` is a 1.5 MB native binary built on `librdkafka` that starts in single-digit milliseconds and pipes cleanly. The killer feature is composable Unix glue: `kcat -C -t orders -e | jq 'select(.amount>1000)' | kcat -P -t large-orders` does cross-topic filtering in one line; `-f '%p\t%o\t%k\t%s\n'` gives `awk`-friendly output with partition, offset, key, payload columns; `-Z` distinguishes null payloads from empty strings (the bane of CDC pipelines). For SRE work, `kcat -Q -t topic:partition:timestamp` resolves "what was the offset at 14:30 UTC" in one round-trip, which is otherwise a multi-step admin-API dance.

## Niche category
Kafka protocol CLI — daemonless, JVM-free producer/consumer/metadata tool

## Repo
https://github.com/edenhill/kcat

## Version pinned
`1.7.0`

## License
- SPDX: `BSD-2-Clause`
- License file in upstream repo: `LICENSE`

## Install
```sh
# macOS
brew install kcat

# Debian / Ubuntu
apt-get install kafkacat   # historical package name; binary is `kcat` post-1.7

# From source (requires librdkafka)
git clone https://github.com/edenhill/kcat && cd kcat && ./bootstrap.sh
```

## Usage
```sh
# Produce JSON lines from stdin into a topic with a key per line
jq -c . events.ndjson | kcat -P -b broker:9092 -t events -K: -l

# Consume from a specific timestamp, decode Avro via schema registry, emit one JSON-per-line
kcat -C -b broker:9092 -t orders -o s@1700000000000 -e \
  -s value=avro -r http://schema-registry:8081 -f '%s\n'

# Cluster metadata as JSON for piping into jq / monitoring
kcat -b broker:9092 -L -J | jq '.topics[] | {name, partitions: (.partitions|length)}'

# Tail a topic with key + offset + partition columns
kcat -C -b broker:9092 -t logs -f '[%p@%o] %k => %s\n'
```

## When to pick `kcat` vs alternatives
- **vs `kafka-console-producer.sh` / `kafka-console-consumer.sh`**: kcat is two orders of magnitude faster to start, has better CLI ergonomics (`-f` format strings, `-J` JSON output), and doesn't require a JDK. The official scripts win only when you need a feature `librdkafka` hasn't shipped yet (rare).
- **vs `kaf`** (Go, [birdayz/kaf](https://github.com/birdayz/kaf)): `kaf` has nicer interactive UX (`kaf consume topic` with auto-config), but kcat's `-f` format strings + `-Z` null handling + libserdes-backed Avro/Protobuf decode are unmatched for shell pipelines and CDC debugging.
- **vs Redpanda's `rpk topic consume`**: `rpk` is excellent if you're already on Redpanda Cloud (auth and broker discovery are zero-config), but it bundles many other commands; kcat is the one-tool-one-job choice for vendor-neutral Kafka protocol work.
- **vs writing a Python `confluent-kafka` script**: use kcat for ad-hoc inspection, one-liners, and shell-pipeline transforms; reach for the library when you need exactly-once semantics, transactional producers, or a long-running consumer group with cooperative rebalancing.
