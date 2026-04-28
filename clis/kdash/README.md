# kdash

> **A simple and fast dashboard for Kubernetes** — a Rust TUI that
> gives you a single-pane-of-glass view of your cluster's nodes,
> pods, services, deployments, and metrics without leaving the
> terminal. Pinned to **v1.1.1** (commit
> `07c6b19a5cf17fced56f4ea50a7c442f863fdf3c`,
> [LICENSE](https://github.com/kdash-rs/kdash/blob/main/LICENSE),
> MIT).

Source: <https://github.com/kdash-rs/kdash>

## TL;DR

`kdash` is what you reach for when you have ten browser tabs open on
the cluster dashboard and you just want to see, in one keystroke,
"which nodes are pressured, which pods are crash-looping, and what is
my CPU/memory headroom right now". It is a single Rust binary that
talks to your current kubeconfig context, polls the API + metrics
server on a tunable interval (`-t 5` for five-second refresh), and
renders a multi-tab `tui-rs` interface: an **Overview** tab with node
utilisation bars + pod status histogram + namespace pivot, plus
dedicated tabs for Pods, Services, Nodes, ConfigMaps, StatefulSets,
ReplicaSets, Deployments, Jobs, DaemonSets, CronJobs, Secrets, and
Ingresses. Selecting a row gives you `d` (describe), `l` (logs,
streamed live), `s` (shell into the container), `y` (dump the YAML),
and `?` for the keymap. Context switching is built in — `c` opens a
picker over every context in your kubeconfig — and namespace
switching with `n` filters the whole UI.

## Install

```bash
# Homebrew (macOS / Linux)
brew tap kdash-rs/kdash && brew install kdash

# Cargo
cargo install --locked kdash

# Docker
docker run --rm -it -v ~/.kube/config:/root/.kube/config deepu105/kdash
```

```bash
# Use a non-default context
kdash --context my-prod-cluster

# Tune the refresh tick (seconds)
kdash -t 2
```

## Niche

The "**read-only, multi-resource, single-binary cluster dashboard**"
slot. Where [`k9s`](../k9s/) is the heavyweight power tool — every
Kubernetes resource type, plugin system, port-forwarding,
benchmarking, RBAC viewer, plus a config DSL — `kdash` is the
opposite trade-off: fewer resource types, no plugins, no edit/apply
flows worth speaking of, but a *single overview screen* that puts node
pressure + pod health + cluster utilisation on one tab. It is the tool
you keep in a corner tmux pane during an incident; `k9s` is the tool
you full-screen when you need to actually fix something.

## Why it matters

- **Cluster overview tab** — node CPU/memory utilisation, pod count by
  status, and namespace breakdown on one screen. Most TUIs are
  resource-list-first; kdash is summary-first.
- **Live log streaming with one keystroke** — `l` on any pod tails
  logs through a scrollback; no `kubectl logs -f` ceremony, no copy-
  paste of the pod name, container picker shows up inline if the pod
  has more than one.
- **Context + namespace pickers built in** — `c` and `n` open fuzzy
  pickers over your kubeconfig contexts and the cluster's namespaces;
  no shelling out to `kubectx` / `kubens`.
- **MIT, single Rust binary, ~10 MB** — no Helm chart, no operator,
  no in-cluster footprint; reads your local kubeconfig and the
  Kubernetes API directly. Optional metrics-server dependency for the
  utilisation bars; degrades gracefully if it is missing.
