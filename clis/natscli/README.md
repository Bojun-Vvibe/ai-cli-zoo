# natscli

> **The official command-line client for the NATS messaging system** —
> publish / subscribe to subjects, manage JetStream streams + consumers
> + key-value buckets + object stores, run benchmarks, and operate a
> NATS cluster end-to-end from one Go binary. Pinned to **v0.3.2**
> ([LICENSE](https://github.com/nats-io/natscli/blob/main/LICENSE),
> Apache-2.0).

Source: <https://github.com/nats-io/natscli>

## TL;DR

`nats` is the daily-driver CLI for everyone who runs or builds against a
NATS server: it speaks plain pub/sub on the core protocol, but the
interesting surface is **JetStream** — NATS's at-least-once persistence
layer with streams, durable consumers, work-queue and key-value and
object-store APIs all expressed as subjects. One static binary lets you
create a stream (`nats stream add`), publish into it (`nats pub`),
attach a durable consumer (`nats consumer add`), pull messages
(`nats consumer next`), inspect lag (`nats stream report`), and
benchmark throughput (`nats bench`) without writing a single line of
Go / Python / TS client code. The same binary administers a multi-node
cluster: `nats server check`, `nats server report jetstream`,
`nats server list`, `nats account info`. Configuration is contexts
(`nats context add prod --server=... --creds=...`) so switching
between dev / staging / prod is one flag.

## Install

```bash
# Homebrew (macOS / Linux)
brew tap nats-io/nats-tools
brew install nats-io/nats-tools/nats

# Go (any platform with Go >=1.22)
go install github.com/nats-io/natscli/nats@v0.3.2

# Direct binary download
curl -sf https://binaries.nats.dev/nats-io/natscli/nats@v0.3.2 | sh

# verify
nats --version    # 0.3.2
```

## License

Apache-2.0 — see
[LICENSE](https://github.com/nats-io/natscli/blob/main/LICENSE).
Permissive, redistributable, vendor-friendly. Same license as the
`nats-server` itself and the official client SDKs (`nats.go`,
`nats.py`, `nats.deno`, `nats.rs`), so an end-to-end NATS stack —
broker + CLI + clients — is uniformly Apache-2.0.

## One Concrete Example

```bash
# 1. point at a server (saved as a named context for later)
nats context add local --server=nats://127.0.0.1:4222 --select

# 2. core pub/sub smoke test (two terminals)
nats sub "orders.>"                                      # terminal A
nats pub orders.created '{"id":42,"sku":"A"}'            # terminal B

# 3. create a JetStream stream that captures everything on orders.>
nats stream add ORDERS --subjects "orders.>" --storage file \
    --retention limits --max-age 7d --max-bytes 10GB \
    --replicas 1 --discard old --dupe-window 2m

# 4. publish persistent messages (acked by the stream)
nats pub orders.created '{"id":43}' --count 1000

# 5. add a durable pull consumer and process messages
nats consumer add ORDERS workers --pull --deliver all \
    --ack explicit --max-deliver 5 --filter "orders.created"
nats consumer next ORDERS workers --count 10 --ack

# 6. stream introspection
nats stream report                  # per-stream msgs / bytes / consumers
nats consumer report ORDERS         # per-consumer lag + ack-pending

# 7. KV bucket as a subject-backed map
nats kv add CONFIG --history 5
nats kv put CONFIG feature.flag.x true
nats kv get CONFIG feature.flag.x
nats kv watch CONFIG                # stream all updates as they happen

# 8. quick load test against the local server
nats bench foo --pub 4 --sub 4 --msgs 1000000 --size 256
```

## Niche It Fills

**The single binary that lets you operate, debug, and benchmark a NATS
deployment without writing client code.** The NATS broker
(`nats-server`) ships only the daemon; everything operational —
"create this stream", "what's the consumer lag", "republish these 1000
messages from a backup", "is this cluster healthy", "benchmark this
subject at 50k msg/s" — happens through `nats`. Without it you write
a Go program for every operation.

## Why use it

1. **JetStream is administered as subjects, not as a separate API
   surface.** `nats stream add ORDERS --subjects "orders.>"` and the
   stream captures every matching publish on the wire — no producer
   code change, no separate "JetStream client". The same publishers
   keep using `nats.Conn.Publish()`; persistence becomes a server-side
   concern toggled by adding or removing a stream.
2. **Contexts make multi-environment ops safe.** `nats context add
   prod --server=tls://prod.example:4222 --creds=./prod.creds` then
   `nats context select prod`. Every subsequent command targets prod
   without `--server` flags, and `nats context show` prints the
   current target so you always know which environment you're about
   to mutate.
3. **`nats bench` is a real load generator, not a toy.** Coordinated
   pub + sub workers, message size + count + rate flags, JetStream
   ack-mode benchmarks (`--js --pull`), per-message latency
   histograms, and a CSV output for plotting. The same binary that
   creates the stream measures it.

## Vs Already Cataloged

- **Vs [`grpcurl`](../grpcurl/):** different transport. `grpcurl`
  exercises gRPC services over HTTP/2; `nats` exercises NATS subjects
  over the NATS wire protocol (TCP or WebSocket). Both are "speak
  the protocol from the shell so you can debug without writing client
  code"; pick by what your service exposes.
- **Vs [`websocat`](../websocat/):** websocat is a generic WebSocket
  swiss-army knife. `nats` understands the NATS protocol natively
  (subjects, queue groups, JetStream acks, KV revisions) so you get
  semantic operations instead of raw frames.
- **Vs [`k6`](../k6/) / [`vegeta`](../vegeta/):** those load HTTP.
  `nats bench` loads NATS subjects with the right concurrency model
  for pub/sub (queue-group fan-out, JetStream ack-pending limits).

## Caveats

- **NATS-only.** Won't talk to Kafka, RabbitMQ, Redis Streams, or
  MQTT brokers. For Kafka use `kcat` / `rpk`; for Redis use
  `redis-cli`.
- **JetStream operations require server-side JetStream enabled.**
  `nats-server -js` or `jetstream {}` in the server config. On a
  vanilla `nats-server` the stream / consumer / KV subcommands
  return `JetStream not enabled`.
- **Credentials handling is your responsibility.** `--creds` files
  contain JWTs that grant publish/subscribe rights — treat them
  like SSH keys, not like a `--password` flag. NATS recommends
  per-environment, scoped credentials rather than a single root
  user.
- **Version cadence is decoupled from `nats-server`.** The CLI and
  server release on independent schedules; an older `nats` against
  a newer server usually works (forward-compat) but new server
  features (e.g., new consumer flags) take a CLI release to surface.
