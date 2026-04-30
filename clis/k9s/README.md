# k9s

> **Terminal UI for Kubernetes clusters.** Full-screen TUI that
> turns `kubectl get / describe / logs / exec / edit / delete`
> into a keyboard-driven console: live-updating resource lists
> across all namespaces, drill-down into pods → containers →
> logs / shell, port-forward / cordon / drain / scale by
> hotkey, plugin system for custom actions, and pluggable
> "skins" + benchmark / pulse / xray views for cluster health.
> Pinned to **v0.50.18**
> ([LICENSE](https://github.com/derailed/k9s/blob/master/LICENSE),
> Apache-2.0).

Source: <https://github.com/derailed/k9s>

## TL;DR

`k9s` opens against your current kube context and renders any
resource by typing `:<resource>` (`:pods`, `:deploy`, `:svc`,
`:ing`, `:node`, even CRDs) — hotkeys then act on the
highlighted row: `l` logs, `s` shell, `d` describe, `e` edit
in `$EDITOR`, `Ctrl-d` delete, `Shift-f` port-forward. Filter
inline with `/regex`, switch namespace with `:ns`, switch
context with `:ctx`. RBAC-aware (`:rbac` shows what the
current user can do), supports read-only mode for incident
response, and ships a `pulses` dashboard plus `popeye` /
`benchmark` integrations for a one-pane cluster health view.

## Install

```bash
# Homebrew (macOS / Linux)
brew install k9s

# Install script
curl -sS https://webi.sh/k9s | sh

# Go install
go install github.com/derailed/k9s@latest

# Container (read-only kubeconfig mount)
docker run --rm -it \
  -v $HOME/.kube/config:/root/.kube/config:ro \
  derailed/k9s
```

## Example

```bash
# Default: opens against current context, all namespaces if RBAC allows
k9s

# Pin to one namespace + a non-default context
k9s -n payments --context staging-eu

# Read-only mode for an on-call session you do not want to mutate
k9s --readonly

# Headless command mode: jump straight to a resource view
k9s -c deploy
```

## When to use

- You triage Kubernetes incidents and want pod logs, exec,
  and describe one keystroke apart instead of re-typing
  `kubectl` invocations.
- You manage many contexts / namespaces and need fast switching
  with RBAC visibility.
- You want a cluster overview (nodes, pulses, popeye lints,
  pod metrics) without standing up a separate dashboard.

## When NOT to use

- You need scripted automation in CI — k9s is interactive;
  use plain `kubectl` or `kubectl-neat` / `kustomize` there.
- You are looking for GitOps reconciliation or drift detection
  — that is `argocd` / `flux`, not k9s.
- The cluster is locked down to API-only access without an
  interactive TTY (bastion-less serverless setups).
