# natscli

## What it does
The official command-line interface for [NATS](https://nats.io), a high-performance pub/sub + request/reply + persistent-streaming messaging system. The single `nats` binary covers every surface: `nats pub subj msg` / `nats sub subj` for core pub/sub, `nats req subj msg` for request/reply, `nats stream add|ls|info|purge` for JetStream persistent streams, `nats consumer add|next` for durable consumers with at-least-once delivery, `nats kv` for the JetStream-backed key/value store, `nats object` for the S3-like object store, `nats account` / `nats auth` for decentralized JWT/NKey credential management, and `nats bench` for built-in throughput/latency benchmarks.

## Why it's interesting
NATS occupies a niche that no other broker quite fills: sub-millisecond fan-out at millions of msg/s with a single 20 MB Go binary server, *plus* an opt-in persistence layer (JetStream) that gives you Kafka-like ordered streams + consumer groups + replay without a separate cluster. `natscli` is the way you actually exercise this — `nats bench --pub 10 --sub 10 --msgs 1000000 foo` produces a publish/subscribe latency histogram in five seconds; `nats stream report` shows per-stream message counts, byte usage, and consumer lag in a single table; `nats context` lets you save named credential bundles (URL + creds + TLS) and switch between dev/staging/prod with `nats context select prod`. The KV (`nats kv put cfg key val` → `nats kv watch cfg`) and Object (`nats object put bucket file.bin`) subcommands turn the broker into a Consul/etcd + minimal-S3 substitute for edge deployments where running three separate stateful systems is overkill.

## Niche category
NATS messaging CLI — pub/sub, JetStream streams, KV, object store, account management

## Repo
https://github.com/nats-io/natscli

## Version pinned
`v0.3.2`

## License
- SPDX: `Apache-2.0`
- License file in upstream repo: `LICENSE`

## Install
```sh
# macOS
brew tap nats-io/nats-tools && brew install nats-io/nats-tools/nats

# Linux / generic
curl -sf https://binaries.nats.dev/nats-io/natscli/nats@latest | sh

# Go install
go install github.com/nats-io/natscli/nats@latest
```

## Usage
```sh
# Save a named connection context (URL, creds, TLS) and select it
nats context save prod --server tls://nats.prod:4222 --creds ~/.nats/prod.creds
nats context select prod

# Core pub/sub — one terminal subscribes, another publishes
nats sub 'orders.>'                       # wildcard subscription
nats pub orders.new '{"id":1,"amt":42}'

# Request/reply with a 2-second timeout
nats req svc.echo "ping" --timeout 2s

# Create a JetStream stream + durable consumer, then pull messages
nats stream add ORDERS --subjects 'orders.>' --storage file --replicas 3
nats consumer add ORDERS workers --pull --ack explicit --deliver all
nats consumer next ORDERS workers --count 100 --ack

# Built-in latency + throughput benchmark (10 pub × 10 sub, 1M msgs)
nats bench foo --pub 10 --sub 10 --msgs 1000000 --size 256
```

## When to pick `natscli` vs alternatives
- **vs Kafka tooling ([`kcat`](../kcat/), `rpk`)**: pick NATS + natscli when you need sub-millisecond fan-out, ephemeral pub/sub *and* persistent streams from one binary, and decentralized JWT-based auth. Kafka wins for very-high-retention log workloads (TB-scale) and the broader connector ecosystem (Debezium, Connect).
- **vs MQTT brokers (`mosquitto_pub`/`sub`)**: natscli covers the same ad-hoc pub/sub use case but adds request/reply, JetStream persistence, KV, and object store in one tool. MQTT is still the right answer when you're talking to constrained IoT devices that already speak the protocol.
- **vs Redis Streams / `redis-cli xadd`**: NATS JetStream gives you proper consumer groups with at-least-once + explicit ack and per-message TTL across a clustered store; Redis Streams is simpler if you already run Redis and don't need multi-node replication of the stream itself.
- **vs the language SDKs (`nats.go`, `nats.py`)**: use `natscli` for operations, debugging, benchmarking, and one-off scripts. Reach for the SDK when you need long-lived consumers with backoff, custom dispatchers, or to embed NATS connectivity into a service.
