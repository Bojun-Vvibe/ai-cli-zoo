# kail

> **A Kubernetes log-tailing CLI that streams logs from
> *every* matching pod across a cluster, with rich
> selectors for namespace / labels / service / ingress /
> deployment / node / container** — a single Go binary that
> talks to the Kubernetes API, watches for pod additions /
> removals while it runs (so a pod that starts mid-tail
> joins the stream and a pod that dies drops out cleanly),
> and prefixes each line with a colored
> `<namespace>/<pod>[<container>]:` so multiplexed output
> stays legible. Pinned to **v0.17.4**
> ([LICENSE.txt](https://github.com/boz/kail/blob/master/LICENSE.txt),
> MIT).

Source: <https://github.com/boz/kail>

## TL;DR

`kubectl logs -f` only follows one pod (or one container in
one pod) at a time. The instant your service runs more than
one replica — which is most of the time — you are stuck
either looping `kubectl logs` over `kubectl get pods -o name`
in a shell script, opening N terminals, or reaching for a
log-aggregation backend (Loki, Elasticsearch). `kail` is the
small fix that lives between "no aggregation" and "deploy
Loki": one binary, no cluster-side agent, that resolves any
Kubernetes selector you care about (label, service, ingress,
deployment, statefulset, daemonset, replicaset, job, node,
namespace, container name) into the matching pod set,
streams logs from all of them, and keeps the set live as
pods come and go.

## Install

```bash
# Homebrew (macOS / Linux)
brew install kail

# Pre-built release tarball
curl -LO https://github.com/boz/kail/releases/download/v0.17.4/kail_0.17.4_darwin_arm64.tar.gz
tar xf kail_0.17.4_darwin_arm64.tar.gz
sudo install kail /usr/local/bin/

# Linux x86_64
curl -L https://github.com/boz/kail/releases/download/v0.17.4/kail_0.17.4_linux_amd64.tar.gz \
    | tar xz
sudo install kail /usr/local/bin/

# Go install (head, not pinned)
go install github.com/boz/kail/cmd/kail@v0.17.4

# Run it inside the cluster as a pod (one-shot)
kubectl run kail --image=abozanich/kail:v0.17.4 --rm -it --restart=Never -- --ns default

# verify
kail --version    # 0.17.4
```

`kail` reads `~/.kube/config` like every other Kubernetes CLI,
honors `KUBECONFIG`, and respects the current context (`kubectl
config current-context`); pass `--context` / `--cluster` to
override per invocation. RBAC: needs `pods/log` `get` and
`pods` `list`/`watch` in the namespaces it tails.

## Use it for

```bash
# Tail every pod in the current namespace
kail

# Tail every pod cluster-wide (RBAC permitting)
kail --ns ""

# Tail one namespace
kail --ns kube-system

# Tail by label selector (this is the everyday usage)
kail --label app=nginx
kail --label "app in (api,worker),env=prod"

# Tail by Kubernetes object — kail resolves it to the pod set
kail --svc nginx                  # pods backing a Service
kail --deploy api-server          # pods of a Deployment
kail --ds fluentd                 # pods of a DaemonSet
kail --sts postgres               # pods of a StatefulSet
kail --job nightly-backup         # pods of a Job
kail --rs api-server-7d4f         # pods of a ReplicaSet

# Tail an ingress's backend pods (great for debugging an URL)
kail --ing my-ingress

# Tail every pod scheduled on one node
kail --node worker-3

# Restrict to one container name across all matched pods
kail --label app=api --containers=api-server

# Filter by ignoring noisy namespaces
kail --ignore-ns kube-system,monitoring

# Combine selectors (intersection)
kail --ns prod --label app=api --containers=app

# Pipe somewhere structured
kail --label app=api --since 1h | tspin
kail --label app=api --output json | jq 'select(.message | contains("ERROR"))'

# One-shot recent slurp instead of follow
kail --label app=api --since 10m --no-follow > slice.log
```

`--since 5m` is the kubectl-equivalent of "go back 5 minutes
before tailing"; `--dry-run` prints the matched pod set and
exits, which is the fastest way to validate a label
selector. The default output prefix
`<ns>/<pod>[<container>]:` is colored per-pod (stable hash),
so a multiplexed stream from 30 pods is still scannable.

## Why include it in a CLI catalog

1. **It is the missing 80%-case between `kubectl logs -f` and
   a log-aggregation cluster.** Almost every team running
   more than two replicas of anything wants "tail logs from
   all pods of my service" without first deploying Loki +
   Promtail + Grafana. `kail` is the one binary that does
   exactly that and nothing more, with no cluster-side
   install footprint (it only needs an RBAC role).
2. **The selector surface is the differentiator.** `kubectl
   logs -l app=foo --all-containers --max-log-requests=20
   -f` exists in modern `kubectl` and covers labels, but
   `kail`'s `--svc / --ing / --deploy / --node / --job`
   selectors resolve a Kubernetes *object* to its current
   pod set and re-resolve as the set changes. Tailing "every
   pod backing this Service" without manually copy-pasting
   the Service's selector is the quality-of-life jump.
3. **Live pod set, not a snapshot.** Run `kail --label
   app=api` while a deploy rolls out and the new pods join
   the stream the moment they pass the readiness probe;
   terminating pods drop off without dumping a "pod not
   found" error into your terminal. That makes it the right
   tool for "watch the logs *during* a rolling update".

For an LLM-CLI workflow, `kail --since 1h --no-follow
--label app=api > /tmp/recent.log` is the canonical "give
me the last hour of logs from my service as one file" recipe
— much smaller than asking the agent to enumerate pods and
loop `kubectl logs`. With `--output json`, each line is a
structured record (`{ns, pod, container, message}`) that an
agent can `jq`-filter without re-parsing the prefix.

## Vs Already Cataloged

- **Vs [`stern`](../stern/):** closest peer — `stern` is the
  most popular multi-pod log tailer (Wercker → CNCF). The
  two overlap heavily; differences: `stern` matches by
  regex on the pod name (`stern api-.*`), `kail` matches by
  Kubernetes object (`kail --deploy api`). `stern`'s ecosystem
  is broader (Helm chart wrapper, etc.); `kail` is a smaller
  binary and the object-selector model is more precise on
  large clusters where pod names alias. Pick `stern` for
  regex / name-based selection; pick `kail` when you think
  in Services / Deployments / Ingresses.
- **Vs [`kubectl`](../kubectl/) `logs`:** `kubectl logs -l
  selector --all-containers -f` (added in 1.23+) covers the
  label-selector case but does not auto-discover new pods
  mid-stream the same way, does not have ingress / service
  resolution, and tops out at `--max-log-requests` (default
  5 in 1.23, 50 in newer). `kail` is a focused tool when
  the cluster has more than that many replicas of one app.
- **Vs [`k9s`](../k9s/):** orthogonal — `k9s` is a full
  cluster TUI (browse pods, exec, edit YAML, port-forward);
  `kail` is a streaming log multiplexer. They co-exist:
  drive `k9s` to find which Service is misbehaving, then
  `kail --svc that-service` in another pane.
- **Vs [`kail-operator`](https://github.com/boz/kail-operator)
  (same author, not cataloged):** the operator runs `kail`
  inside the cluster and exposes results via WebSocket;
  the CLI here is the local-machine version most users
  want.

## Caveats

- **Last release v0.17.4 shipped 2024-01.** Active enough
  to still be the canonical tool, but development cadence
  is slow; `stern` ships more often. The protocol surface
  (Kubernetes API) is stable, so this matters less than it
  reads.
- **No log-storage / no search.** `kail` is a pipe, not an
  index. Need to grep history? Pipe to a file
  (`kail ... | tee logs.txt`) or use `--since`; for
  retention you still want Loki / OpenSearch / ELK.
- **High pod counts can hit API-server rate limits.**
  Tailing >100 pods opens >100 streaming watches; on shared
  clusters with strict API-server quotas you will get
  throttled. Workaround: scope with `--ns` /
  `--containers` / `--label` to narrow the set.
- **Container restart loses the stream for that container.**
  `kail` re-attaches as the new pod object becomes
  available, but the gap is visible (a few hundred ms). For
  reliable multi-restart capture, ship logs to a backend.
- **No timestamp normalization.** Each pod's log line keeps
  the timestamp the container emitted; if pods write in
  different time zones the multiplexed stream is not
  monotonic. Pipe through `humanlog` to reformat or through
  `tspin` to color the timestamps for visual sorting.
