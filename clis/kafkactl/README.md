# kafkactl

> **Single-binary CLI for Apache Kafka with consumer-group-aware
> defaults, multi-context configs, Avro/Protobuf/JSON-Schema
> support via Schema Registry, and structured `--output yaml`/
> `json` for scripting** — what `kubectl` is to Kubernetes,
> `kafkactl` aims to be for Kafka clusters: one binary, named
> contexts in `~/.config/kafkactl/config.yml`, a verb-noun command
> shape (`create topic`, `produce`, `consume`, `describe consumer-
> group`, `reset offsets`, `alter topic`, `list acl`), and machine-
> parseable output flags. Pinned to **v5.18.0** (released
> 2026-02-12,
> [LICENSE.md](https://github.com/deviceinsight/kafkactl/blob/v5.18.0/LICENSE.md),
> Apache-2.0).

Source: <https://github.com/deviceinsight/kafkactl>

## TL;DR

The Kafka tooling space is split between (a) the upstream
`kafka-*-sh` shell scripts shipped with the broker (verbose,
per-script JVM warmup, no contexts, no schema-registry
integration, output is whatever `println` decided), (b)
[`kcat`](../kcat/) (formerly `kafkacat`, excellent for one-off
produce/consume but not an admin tool — no consumer-group
reset, no topic-config edits, no ACLs), and (c) `kafkactl`,
which fills the "I need to administer this cluster from a
script" gap. Named contexts mean `kafkactl --context prod
describe topic events` is one short command instead of `--
bootstrap-server prod1:9092,prod2:9092 --command-config
/etc/kafka/admin.properties`. Schema Registry support means
Avro / Protobuf / JSON-Schema messages decode automatically on
`consume`. `--output yaml` / `--output json` make every
command a scriptable building block.

## Install

```bash
# macOS / Linux (Homebrew)
brew tap deviceinsight/packages
brew install kafkactl

# Linux: prebuilt .deb / .rpm at the release page
curl -L -o kafkactl.deb \
  https://github.com/deviceinsight/kafkactl/releases/download/v5.18.0/kafkactl_5.18.0_linux_amd64.deb
sudo dpkg -i kafkactl.deb

# Snap
sudo snap install kafkactl

# Go install (any OS with a Go toolchain)
go install github.com/deviceinsight/kafkactl/v5@v5.18.0

# Container (Alpine-based)
docker run --rm -v ~/.config/kafkactl:/root/.config/kafkactl \
  deviceinsight/kafkactl:v5.18.0 get topics

# Verify
kafkactl version          # kafkactl version 5.18.0
```

First-run config — declare contexts in
`~/.config/kafkactl/config.yml`:

```yaml
contexts:
  default:
    brokers:
      - localhost:9092
  prod:
    brokers:
      - prod-kafka-1:9093
      - prod-kafka-2:9093
    tls:
      enabled: true
      ca: /etc/kafka/ca.crt
      cert: /etc/kafka/client.crt
      certKey: /etc/kafka/client.key
    avro:
      schemaRegistry: https://schema-registry.prod:8081
current-context: default
```

## License

Apache-2.0 — see
[LICENSE.md](https://github.com/deviceinsight/kafkactl/blob/v5.18.0/LICENSE.md).
Permissive, patent grant included; redistribute and embed
freely.

## Common invocations

```bash
# Switch contexts the kubectl way
kafkactl config use-context prod
kafkactl config current-context

# Topic admin
kafkactl create topic events --partitions 12 --replication-factor 3 \
  --config retention.ms=604800000 --config compression.type=zstd
kafkactl get topics                                 # list
kafkactl describe topic events                      # config + partitions + ISR
kafkactl alter topic events --config retention.ms=2592000000
kafkactl delete topic deprecated-events

# Produce
kafkactl produce events --key user-42 --value '{"event":"login"}'
echo '{"k":"v"}' | kafkactl produce events --value -
# Avro / Protobuf — auto-encode against Schema Registry
kafkactl produce events --value-schema-version latest \
  --value '{"user_id":42,"event":"login"}'

# Consume
kafkactl consume events                              # tail
kafkactl consume events --from-beginning --exit     # full snapshot, then exit
kafkactl consume events --print-keys --print-headers --print-timestamps
kafkactl consume events --group analytics-debug --offset oldest

# Consumer-group ops — the killer feature vs `kcat`
kafkactl get consumer-groups
kafkactl describe consumer-group analytics
kafkactl reset offsets --group analytics --topic events --to-earliest --execute
kafkactl reset offsets --group analytics --topic events --to-datetime \
  '2026-05-01T00:00:00Z' --execute

# ACLs
kafkactl get acl
kafkactl create acl --topic events --operation read --principal "User:alice" --allow

# Machine-parseable output for scripting
kafkactl get topics -o yaml | yq '.[] | select(.partitions > 16)'
kafkactl get consumer-groups -o json | jq '.[] | select(.state=="Stable")'
```

## Why use it

- **Named contexts.** One `~/.config/kafkactl/config.yml`
  declares `dev`, `staging`, `prod` clusters with their TLS /
  SASL / Schema-Registry settings; `kafkactl --context prod
  ...` is the entire ceremony. The upstream `kafka-*.sh`
  scripts have no equivalent — every invocation re-passes
  `--bootstrap-server` and `--command-config`.
- **Schema Registry is built-in.** `--value-schema-version
  latest` on `produce` encodes Avro / Protobuf / JSON-Schema
  against the registered schema; `consume` decodes
  automatically when the topic uses a schema. `kcat` requires
  pre-encoding the value yourself or piping through `jq +
  avro-tools`.
- **`-o yaml` / `-o json` everywhere.** Every read command
  emits structured output flag-controlled, so `kafkactl get
  topics -o json | jq` and `kafkactl describe consumer-group
  foo -o yaml | yq` are the right shape for any automation.
  Upstream scripts emit free-form text that needs awk/sed
  scraping.
- **Consumer-group reset is first-class.** `kafkactl reset
  offsets --group g --topic t --to-datetime <iso8601>
  --execute` (or `--to-offset`, `--to-earliest`,
  `--to-latest`, `--shift-by N`) is a single command. The
  shell-script equivalent (`kafka-consumer-groups.sh`) works
  but the flag set is opaque and the JVM warmup is a half-
  second tax per invocation.

## Vs Already Cataloged

- **Vs [`kcat`](../kcat/):** complementary, not redundant.
  `kcat` is the surgical tool — produce / consume / metadata-
  query at the broker level, optimised for one-shot
  scripting and pipe-friendly output. Use `kcat` for "pipe
  500 NDJSON lines into topic X" or "tail topic Y until
  match". Use `kafkactl` for the *admin* surface — consumer
  groups, topic configs, ACLs, multi-context configs, schema-
  registry-aware messages — that `kcat` deliberately omits.
- **Vs [`nats`](../nats/) / [`natscli`](../natscli/):** wrong
  bus. `nats` / `natscli` administer NATS / JetStream
  clusters; `kafkactl` administers Kafka clusters. Same
  shape (verb-noun CLI for a streaming broker) but distinct
  protocols and consumer-group models.
- **Vs [`k9s`](../k9s/):** orthogonal. `k9s` is an interactive
  TUI for Kubernetes. `kafkactl` is a non-interactive CLI for
  Kafka. They compose: a `k9s`-managed Kafka StrimziCluster
  in K8s, administered topic-by-topic with `kafkactl` from a
  laptop or CI job.

## Caveats

- **Sarama-based, not librdkafka-based.** `kafkactl` uses the
  Go-native `IBM/sarama` client. For 99% of admin and
  produce/consume workloads this is invisible; for ultra-high-
  throughput producers (>100k msg/sec sustained) the
  `librdkafka`-based `kcat` may be faster. Pick by workload.
- **Schema-Registry features require Confluent-shape
  registry.** Apicurio, Karapace, and Confluent Schema
  Registry are supported (Schema Registry HTTP API + magic-
  byte wire format). Schemas served only via custom
  endpoints will not auto-decode.
- **No interactive TUI.** `kafkactl` is verb-noun + flags,
  not curses. For an interactive Kafka explorer (browse
  topics, tail messages with filters, edit consumer-group
  offsets visually) pair with `kaf-ui` / `redpanda-console`
  / `kafka-ui` (web UIs) — different category.
- **Apache Kafka only.** Despite Redpanda / WarpStream / AWS
  MSK / Confluent Cloud all speaking the Kafka protocol,
  `kafkactl` targets the Apache Kafka admin surface;
  protocol-compatible vendors mostly work but vendor-
  specific extensions (Redpanda transforms, Confluent ksqlDB
  integration) are out of scope.
- **`--execute` is required for destructive ops.** `reset
  offsets`, `delete topic`, `delete acl` print a dry-run
  preview without `--execute` — explicit, but the muscle
  memory from upstream scripts (which act immediately) bites
  newcomers who copy-paste.
