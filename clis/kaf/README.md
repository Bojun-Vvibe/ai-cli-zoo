# kaf

> **Modern, single-binary Kafka CLI written in Go — `kcat`-class
> producer / consumer / metadata client with first-class support for
> *named clusters* (`kaf config add-cluster`), Schema Registry
> (Avro / Protobuf / JSON-Schema decode on the fly), consumer-group
> inspection and offset reset, topic create / alter / delete, and
> human-friendly TTY output instead of `kafka-console-*.sh` JVM
> startup latency.** Niche tag: **observability / messaging /
> streaming-data**. Pinned to **v0.2.14**
> ([LICENSE](https://github.com/birdayz/kaf/blob/master/LICENSE),
> Apache-2.0).

Source: <https://github.com/birdayz/kaf>

## TL;DR

`kaf consume orders -f` tails the `orders` topic with key /
timestamp / partition / offset framing and pretty-printed JSON
values, against whatever cluster `kaf config use-cluster ...` is
pointing at — no `--broker-list`, `--bootstrap-server`, and seven
`--consumer.config` flags every invocation. The config file
(`~/.kaf/config`) holds named clusters with SASL / mTLS / Schema
Registry URL baked in, so switching from `dev` to `staging` to
`prod` is one command. Schema-Registry-aware: `kaf consume topic
--decode-msgpack-keys` or auto-decode of Confluent-framed Avro /
Protobuf / JSON-Schema payloads when the registry is configured.
Group ops (`kaf group ls`, `kaf group describe`, `kaf group
commit --topic foo --partition 0 --offset 12345`) are first-class.

## Install

```bash
# Homebrew (macOS / Linux)
brew install kaf

# Go
go install github.com/birdayz/kaf/cmd/kaf@latest

# Direct binary (Linux x86_64)
curl -L https://github.com/birdayz/kaf/releases/download/v0.2.14/kaf_0.2.14_linux_amd64.tar.gz \
  | tar -xz && sudo mv kaf /usr/local/bin/
```

## Example

```bash
# Register a cluster and switch to it
kaf config add-cluster prod \
  --brokers broker-1:9093,broker-2:9093 \
  --tls --schema-registry-url https://registry.example.com
kaf config use-cluster prod

# Tail a topic, decoding Confluent-Avro values via the registry
kaf consume orders -f --decode-protobuf-type=acme.OrderV2

# Reset a consumer group's offset to a timestamp on one partition
kaf group commit billing-svc --topic invoices --partition 3 \
  --offset 1700000000000 --offset-as-timestamp
```

## When to use

- You ship to or operate Kafka / Redpanda / MSK / Confluent Cloud /
  Strimzi clusters and want a Go-binary CLI that starts in
  milliseconds instead of the bundled `kafka-console-*.sh` scripts
  that need a JVM warm-up per invocation.
- You move between several clusters per day and want named
  contexts (`dev` / `staging` / `prod`) the same way `kubectl`
  has contexts.
- Your topics carry Confluent-framed Avro / Protobuf / JSON-Schema
  and you need the CLI to decode them inline against a Schema
  Registry.

## When NOT to use

- You only ever talk to one cluster from one shell pipeline and
  prefer the `librdkafka`-backed format-string ergonomics of
  [`kcat`](../kcat/) — kcat's `-f '%T %k %s\n'` and binary-payload
  handling remain unmatched for shell glue.
- You need a *broker-side* admin tool (rebalance plans, JBOD
  management, KRaft cluster metadata surgery) — use the bundled
  `kafka-*.sh` scripts or `cruise-control`; kaf is a client-side
  CLI.
- You want a GUI — pick AKHQ, Kafdrop, Conduktor, or Redpanda
  Console.
