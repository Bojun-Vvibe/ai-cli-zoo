# linkerd

- **Repo:** https://github.com/linkerd/linkerd2
- **Version:** edge-26.4.4
- **License:** [LICENSE](https://github.com/linkerd/linkerd2/blob/main/LICENSE) (Apache-2.0)
- **Category:** Service mesh / Kubernetes mTLS + traffic policy CLI

## What it is

`linkerd` is the operator CLI for the Linkerd service mesh — a CNCF graduated,
Rust-based ultralight mesh for Kubernetes. The CLI installs the control plane,
injects the per-pod `linkerd2-proxy` sidecar, runs pre/post-flight checks, and
drives mTLS, traffic-policy, multicluster, and `viz` (Prometheus + Grafana)
extensions. The data-plane proxy is written in Rust on top of Tokio and Hyper,
so per-pod overhead is typically <10 MB RSS and sub-millisecond p99 latency,
which is why teams pick Linkerd over Istio when "make every pod mTLS without
explaining Envoy to anyone" is the actual goal.

## Install

```
curl -fsL https://run.linkerd.io/install-edge | sh
export PATH=$PATH:$HOME/.linkerd2/bin
linkerd version
```

## Basic usage

```
linkerd check --pre                                    # cluster prereqs
linkerd install --crds | kubectl apply -f -            # CRDs
linkerd install | kubectl apply -f -                   # control plane
linkerd check                                          # post-install
kubectl get deploy -n my-app -o yaml \
  | linkerd inject - \
  | kubectl apply -f -                                  # add the sidecar
linkerd viz install | kubectl apply -f -               # metrics extension
linkerd viz dashboard &                                # open the UI
linkerd viz stat deploy -n my-app                      # golden metrics
linkerd authz -n my-app deploy/web                     # who can reach this pod
```

## When to use it

- You want **automatic mTLS, identity, and golden metrics** for every pod in
  a Kubernetes cluster with zero application changes and no Envoy expertise.
- You care about **footprint and latency** — the Rust proxy is dramatically
  lighter than Envoy-based meshes for the same feature set on the hot path.
- You need **multicluster** service mirroring with one CLI command per side
  (`linkerd multicluster link`).
- Skip it if you need WASM filter extensibility, gRPC-Web transcoding, or the
  full Envoy xDS surface — Istio/Kuma/Consul Connect are better matches there.
