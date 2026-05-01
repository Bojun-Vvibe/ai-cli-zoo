# kubie

> **`kubectx` / `kubens` with one critical twist: each context is
> a real subshell, not a global edit of `~/.kube/config`** — every
> `kubie ctx <name>` opens a new shell whose `KUBECONFIG` points
> at an isolated temporary file, so two terminal tabs can sit on
> *different* clusters and namespaces simultaneously without
> stepping on each other. Pinned to **v0.25.1** (released
> 2025-04-26,
> [LICENSE](https://github.com/sbstp/kubie/blob/master/LICENSE),
> GPL-3.0).
>
> Source: <https://github.com/sbstp/kubie>

## TL;DR

Single static Rust binary that fixes the original sin of
`kubectl` config: the active context lives in one file shared by
every shell on the box, so the moment you have two windows open
you are one stray `kubectl apply` away from shipping the staging
manifest into prod. `kubie` solves it by running each context in
its own subshell — your prompt segment shows
`[cluster|namespace]`, `exit` drops you back to the parent
context, and the original `~/.kube/config` is never mutated.
Reads multiple kubeconfig files and EKS / GKE / AKS profiles in
parallel, supports fuzzy picker UI when called bare, and ships
shell completion for bash / zsh / fish.

## Install

```bash
# Homebrew (macOS / Linux)
brew install kubie

# Cargo (any OS, requires Rust toolchain)
cargo install kubie

# Static binary (Linux / macOS / Windows)
# https://github.com/sbstp/kubie/releases

# verify
kubie --version    # kubie 0.25.1
```

## Examples

```bash
# fuzzy-pick a context, drop into a subshell scoped to it
kubie ctx
# -> prompt becomes: [prod-eu-west-1|default] $

# direct: open a subshell on a named context
kubie ctx prod-eu-west-1

# inside that subshell, switch namespace (still scoped to this shell only)
kubie ns kube-system

# exec a single command in a context without entering a subshell
kubie exec staging-us-east-1 default kubectl get pods -A

# fan out the same command across many contexts in parallel
kubie exec '*-prod' default kubectl get nodes -o wide

# nest contexts: open prod in one tab, exit, come back to staging
[staging|app] $ kubie ctx prod
[prod|app] $ exit
[staging|app] $    # <- right where you left off

# point kubie at extra kubeconfig dirs (per-cluster files, vault-rendered configs)
# ~/.kube/kubie.yaml:
#   configs:
#     - "~/.kube/configs/*.yaml"
#     - "~/work/clusters/*/kubeconfig"
```

## Use when

- You routinely have **two or more terminal tabs talking to
  different clusters** (staging in one, prod in another, a kind
  cluster in a third) and you have already nuked the wrong
  resource at least once because some background tab quietly
  switched the global context.
- You manage **dozens of kubeconfig files** (per-cluster issued
  by your platform team, GKE / EKS auth plugins, ephemeral kind /
  k3d clusters) and you do not want to merge-and-pray them into
  one giant `~/.kube/config`. `kubie` reads them as a glob and
  presents one fuzzy picker.
- You want a **prompt that always tells you which cluster +
  namespace this shell is on** without configuring `kube-ps1`,
  starship, oh-my-zsh segments, etc. — kubie injects it for you.
- You need to **fan a one-off command across a class of clusters**
  (`kubie exec '*-prod' …`) and aggregate output without writing
  a for-loop wrapper.
- Pair with [`kubectx`](../kubectx/) (the original; kubie is the
  subshell-isolated successor — pick one, do not run both), [`k9s`](../k9s/)
  / [`kdash`](../kdash/) (TUI per cluster — launch them *inside*
  a kubie subshell so they inherit the right context),
  [`stern`](../stern/) / [`kubeshark`](../kubeshark/)
  (multi-pod log + traffic tools that benefit from a clean
  per-shell `KUBECONFIG`),
  [`kubefwd`](../kubefwd/) / [`telepresence`](../telepresence/)
  (tunnel into a cluster — kubie keeps that tunnel scoped to one
  shell so it cannot bleed).

Skip `kubie` if you only ever talk to one cluster from one
terminal — vanilla `kubectx` / native `kubectl config
use-context` is fine and one less binary. Skip if your team's
tooling assumes the *global* current context (some legacy
controllers, some IDE plugins) — kubie's per-shell isolation is
the whole point and will surprise those tools. Not a replacement
for full session managers like
[`kubeswitch`](https://github.com/danielfoehrKn/kubeswitch); the
sweet spot is "kubectx but each switch is a subshell".
