# kubeswitch

> **Drop-in `kubectx` replacement built for fleets of
> hundreds of kubeconfigs**, with pluggable backends
> (filesystem trees, Vault, GKE, EKS, AKS, Rancher,
> OVH, Gardener, plugin scripts), an indexed search
> store, async pre-fetch, and per-shell isolation so
> two terminals never silently share the active
> context. Pinned to **0.9.3**
> ([LICENSE](https://github.com/danielfoehrKn/kubeswitch/blob/master/LICENSE),
> Apache-2.0).

Source: <https://github.com/danielfoehrKn/kubeswitch>

## TL;DR

`kubeswitch` (binary: `switch`, alias `switcher`) is
what you reach for when `kubectx` runs out of road —
specifically when "all clusters in one merged
`~/.kube/config`" stops being feasible because (a) the
file is hundreds of MB, (b) the contexts come from
several sources you do not control (a directory of
files dropped by your platform team, secrets in Vault,
the GKE / EKS / AKS APIs, a Gardener landscape, a
custom inventory script), or (c) you keep clobbering
the active context across terminal tabs. `switch` walks
all configured **stores** in parallel, builds a
searchable index of every context across all of them
(`switch search` + fuzzy filter), and on selection
writes a **per-shell temporary kubeconfig** that
`KUBECONFIG` points at — so the parent shell, your
other tabs, and the on-disk source files are never
mutated. Async cache + hot reload mean a 4000-context
landscape stays sub-second to filter; the hook system
(`switch hooks`) runs custom scripts on cluster switch
(refresh OIDC token, set `KUBE_NAMESPACE`, post a
Slack note); and store plugins make "list me every
cluster I have access to across these 6 places" one
command instead of six.

## When to use

- You manage **dozens to thousands of kubeconfigs**
  from heterogeneous sources (filesystem trees, Vault,
  cloud provider APIs, custom inventory scripts) and
  merging them all into one `~/.kube/config` is no
  longer practical.
- You routinely keep **multiple terminals open against
  different clusters** and need each shell's context
  to be isolated from the others.
- You want **fuzzy search across every context you
  have access to**, not just the ones you have already
  added to a config file.

## When NOT to use

- You only ever talk to **a handful of clusters from
  one merged kubeconfig** — vanilla
  [`kubectx`](../kubectx/) is simpler and ships in
  every package manager.
- You want **per-shell isolation but only from one
  source** — [`kubie`](../kubie/) is the lighter
  Rust-binary answer (subshell + glob over kubeconfig
  files, no plugin store layer).
- You want a **full-screen cluster TUI** that also
  drives `get` / `describe` / `logs` / `exec` —
  [`k9s`](../k9s/) / [`kdash`](../kdash/) own that
  niche; switch only manages the "which context".

## Install

```bash
# Homebrew (tap)
brew install danielfoehrkn/switch/switch

# Go install
go install github.com/danielfoehrKn/kubeswitch@latest

# prebuilt binaries
# https://github.com/danielfoehrKn/kubeswitch/releases

# the recommended shell wrapper (so KUBECONFIG sticks
# in the current shell), eg. for zsh:
source <(switcher init zsh)

# verify
switch version    # 0.9.3
```

## Basic usage

```bash
# fuzzy-pick across every context the configured stores expose
switch

# search by substring across all stores
switch search prod-eu

# previous context (kubectx-style)
switch -

# list / set namespace inside the picked context
switch namespace
switch namespace payments

# manage stores in ~/.kube/switch-config.yaml (filesystem
# tree at ~/.kube/configs by default; add Vault / GKE /
# EKS / AKS / Gardener / Rancher / OVH / plugin entries)
$EDITOR ~/.kube/switch-config.yaml

# refresh the search index after adding a store
switch clean
```

## Pairs with

- [`k9s`](../k9s/) / [`kdash`](../kdash/) — launch
  inside a `switch`-set shell so they inherit the
  isolated `KUBECONFIG`.
- [`stern`](../stern/) / [`kubefwd`](../kubefwd/) —
  multi-pod log + port-forward tools that benefit from
  a clean per-shell context.
- [`kubectx`](../kubectx/) / [`kubie`](../kubie/) —
  the simpler peers in this niche; pick switch when
  store plugins or the indexed search matter.
- [`kubeseal`](../kubeseal/) / [`vault`](../vault/) —
  switch's Vault store reads kubeconfigs straight from
  Vault paths.
