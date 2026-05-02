# nats

> **Official command-line client for NATS** — connect to a NATS
> server or cluster to publish, subscribe, run request/reply RPC,
> manage JetStream streams and consumers, drive Key-Value and
> Object stores, and operate the server (accounts, users,
> leafnodes, monitoring). Pinned to **v0.4.0** (released
> 2026-05-01,
> [LICENSE](https://github.com/nats-io/natscli/blob/main/LICENSE),
> Apache-2.0).

Source: <https://github.com/nats-io/natscli>

## TL;DR

`nats` is the one binary you need to actually *use* and *operate*
a NATS deployment from the terminal. Core messaging is one flag
(`nats pub foo hello` / `nats sub foo`), request/reply is
first-class (`nats reply 'time' 'date'` + `nats request time ''`),
and the same tool drops you straight into JetStream — the
persistent, replicated, exactly-once-capable layer — with
`nats stream add`, `nats consumer add`, `nats stream backup`, and
`nats stream replay`. It also exposes the KV (`nats kv`) and
Object Store (`nats object`) APIs as plain CLI verbs, runs latency
and throughput benchmarks (`nats bench`), and has an operator
surface (`nats server report`, `nats server check`,
`nats account info`) so you can debug a cluster without writing
any Go.

## Install

```bash
# Homebrew (macOS / Linux)
brew tap nats-io/nats-tools
brew install nats

# Go install (any OS with Go 1.22+)
go install github.com/nats-io/natscli/nats@latest

# Static binary (Linux / macOS / Windows / FreeBSD)
# https://github.com/nats-io/natscli/releases

# Docker
docker run --rm -it natsio/nats-box:latest nats --help

# verify
nats --version    # 0.4.0
```

## Examples

```bash
# point the CLI at a server (or a named context for repeated use)
nats context add prod --server nats://nats.example.com:4222 --select

# core pub/sub
nats sub 'orders.>'           # in one terminal
nats pub orders.created '{"id":42}' --reply

# request / reply RPC
nats reply 'svc.time' "$(date -u +%FT%TZ)" &
nats request svc.time '' --timeout 1s

# JetStream: create a persistent stream + a durable consumer
nats stream add ORDERS --subjects 'orders.>' --storage file \
  --replicas 3 --retention limits --max-age 168h --defaults
nats consumer add ORDERS workers --pull --ack explicit --deliver all --defaults
nats consumer next ORDERS workers --count 10

# back up / restore a JetStream stream (disaster recovery)
nats stream backup ORDERS ./orders-backup
nats stream restore ./orders-backup

# Key-Value and Object stores
nats kv add CONFIG --history 5
nats kv put CONFIG feature.x true
nats kv watch CONFIG

# operator: cluster health + per-server report
nats server check connection --connect-warn 50ms
nats server report jetstream
nats server report connections --account SYS

# benchmark: 4 publishers + 4 subscribers, 1M msgs of 1 KB
nats bench foo --pub 4 --sub 4 --msgs 1000000 --size 1024
```

## Use when

- You run NATS (core or JetStream) and want one tool that covers
  *both* application-side messaging and operator tasks — no
  separate "admin" binary.
- You need to debug "is my message getting through?" in
  production: `nats sub`, `nats request`, and `nats stream view`
  beat reading server logs.
- You operate JetStream and need streams, consumers, KV, and
  Object Store as scriptable verbs (CI, runbooks, IaC wrappers)
  rather than as REST calls you have to hand-roll.
- You want a built-in load generator (`nats bench`) that uses the
  same client library as your apps, so the numbers actually map
  to what production sees.

Skip `nats` for "I am evaluating a message bus and have not
chosen one yet" — pick the broker first. Once NATS is in the
stack, `nats` is non-optional: it is what the NATS team itself
uses to inspect a cluster.
