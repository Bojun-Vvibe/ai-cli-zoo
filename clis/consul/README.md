# consul

- **Repo:** https://github.com/hashicorp/consul
- **Version:** v1.22.7
- **License:** [LICENSE](https://github.com/hashicorp/consul/blob/main/LICENSE) (BUSL-1.1)
- **Category:** Service networking / Service discovery + KV + mesh

## What it is

Consul is HashiCorp's service-networking control plane. The `consul` binary
is at once the agent that joins each node to a gossip-based cluster, the
client that talks to the HTTP/DNS API, and the operator tool for running
Raft, ACLs, intentions, and Connect/mesh policy. A single static Go binary
gives you service registry, health checking, distributed KV, multi-DC WAN
federation, and an mTLS service mesh you can drive without standing up a
separate controller.

## Install

```
brew install consul                       # macOS
# or grab the static binary from https://releases.hashicorp.com/consul/
```

## Basic usage

```
consul agent -dev                                         # one-node dev cluster on :8500
consul members                                            # list cluster members
consul services register -name=web -port=8080             # register a local service
consul kv put config/web/log_level info                   # write to the KV store
consul kv get -recurse config/                            # read it back
consul connect proxy -service web -upstream db:9090       # spin up a sidecar
consul intention create -allow web db                     # mesh policy
```

## When to use it

- You want **service discovery + health checks + KV + mTLS mesh** behind one
  binary and one HTTP/DNS API rather than gluing four separate systems.
- You need **multi-datacenter** federation (WAN gossip, cross-DC service
  resolution, prepared queries) without rolling your own control plane.
- You already run Nomad or want a non-Kubernetes mesh — Consul Connect plugs
  straight into VM, bare-metal, and container workloads via Envoy sidecars.
- Skip it if you only need DNS-based discovery inside one Kubernetes cluster
  (CoreDNS + Service is enough) or if BUSL-1.1 is a non-starter for your
  legal review — the OpenBao/etcd combo is the usual permissive fallback.
