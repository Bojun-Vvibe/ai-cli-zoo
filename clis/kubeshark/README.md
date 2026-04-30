# kubeshark

- **Repository:** https://github.com/kubeshark/kubeshark
- **Latest version:** v53.2.3
- **License:** Apache-2.0 — verified at [`LICENSE`](https://github.com/kubeshark/kubeshark/blob/master/LICENSE) (raw: https://raw.githubusercontent.com/kubeshark/kubeshark/master/LICENSE)
- **Niche:** Kubernetes runtime observability / API traffic inspection

## What it does

`kubeshark` is an API-traffic analyzer for Kubernetes — think
"Wireshark for clusters". It deploys a per-node worker (eBPF + libpcap)
that captures L4/L7 traffic flowing between pods and reconstructs it
into protocol-aware sessions: HTTP/1.x, HTTP/2, gRPC, AMQP, Kafka,
Redis, DNS, and more. The CLI tunnels a local UI where you can filter,
replay, and diff requests across services.

```
kubeshark tap                       # tap traffic in the current namespace
kubeshark tap -n my-ns "frontend-*" # restrict to matching pods
kubeshark console                   # open the UI against the running tap
kubeshark clean                     # tear down the worker DaemonSet
```

## Why interesting

Most Kubernetes "observability" tools require you to instrument your
code (OpenTelemetry SDKs, sidecars, mesh injection). `kubeshark` is the
opposite: zero code changes, zero sidecar, zero mesh. You drop it into a
cluster, get an immediate ground-truth view of what services are
*actually* saying to each other on the wire, and tear it back down.

That makes it disproportionately useful in three situations CLI catalogs
usually under-serve:

1. **Reverse-engineering a system you didn't build** — agent-driven
   refactors, vendor handoffs, or migrating off an opaque legacy
   service.
2. **Debugging contract drift** — when a downstream client claims the
   API "changed" but no one knows what shape it's actually sending.
3. **Auditing what an autonomous agent fleet is doing in-cluster**
   without trusting the agents' own logs.

## Caveats

The capture worker is privileged and runs as a DaemonSet, so don't drop
it on a hostile multi-tenant cluster casually. For long-term capture
there's also a paid Pro tier; the OSS CLI here is the on-demand tap.
