# argocd

> **GitOps continuous-delivery CLI for Kubernetes.** The
> client-side companion to the Argo CD controller — manages
> `Application`, `ApplicationSet`, `AppProject`, and cluster /
> repo registrations from your terminal, with sync, diff,
> rollback, history, and live-resource inspection. Pinned to
> **v3.3.8** ([LICENSE](https://github.com/argoproj/argo-cd/blob/v3.3.8/LICENSE),
> Apache-2.0).

Source: <https://github.com/argoproj/argo-cd>

## TL;DR

`argocd` is the operator-facing tip of the Argo CD GitOps
controller: every cluster state mutation lives as an
`Application` CR pointing at a git path (Helm chart, Kustomize
overlay, plain manifests, or a Jsonnet/CMP plugin), and the
controller continuously reconciles cluster against git. The CLI
gives you `app sync`, `app diff` (cluster vs desired), `app
history` + `app rollback <rev>`, `app wait` for healthy/synced
gating in CI, and `app set` to override target revision /
parameters without editing git. `argocd login <server>` stores
context in `~/.config/argocd/config`; from there `argocd cluster
add <kube-context>` registers a managed cluster and `argocd
repo add <url>` wires a source repository (HTTPS token, SSH key,
GitHub App, or OCI Helm).

## Install

```bash
# Homebrew (macOS / Linux)
brew install argocd

# Direct binary (Linux amd64)
curl -sSL -o /usr/local/bin/argocd \
  https://github.com/argoproj/argo-cd/releases/download/v3.3.8/argocd-linux-amd64
chmod +x /usr/local/bin/argocd

# Go install (head)
go install github.com/argoproj/argo-cd/v3/cmd/argocd@v3.3.8
```

## Example

```bash
# One-shot login against an in-cluster Argo CD
argocd login argocd.example.com --sso

# Register a repo and create an app from a Helm chart path
argocd repo add https://github.com/acme/manifests.git --username git --password "$TOKEN"
argocd app create web \
  --repo https://github.com/acme/manifests.git \
  --path charts/web --revision main \
  --dest-server https://kubernetes.default.svc --dest-namespace web \
  --sync-policy automated --self-heal

# CI gate: sync, then block until healthy or fail
argocd app sync web && argocd app wait web --health --timeout 300

# Diff cluster against git, then roll back two revisions
argocd app diff web
argocd app rollback web 42
```

## When to use

- You already run Argo CD in-cluster and want scriptable
  sync / diff / wait gates in CI without poking the API directly.
- You need `app diff` against the live cluster as a PR-time
  guardrail (drift detection, unexpected field deltas).
- You want one binary to manage `Application` CRs across many
  clusters from a single login context.

## When NOT to use

- You have not deployed the Argo CD controller — the CLI alone
  does nothing; reach for plain `kubectl apply -k` or
  [`flux`](../flux/) (different controller) instead.
- You want push-based CD (`kubectl apply` from CI) — that is
  exactly what GitOps replaces; pick a plain `kubectl` /
  `helmfile` pipeline if the pull-based reconciliation loop is
  not what you want.
- You only need one-shot manifest rendering — `helm template` /
  `kustomize build` are simpler and have no controller dependency.
