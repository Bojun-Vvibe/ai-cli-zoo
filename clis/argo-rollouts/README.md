# argo-rollouts

> **Progressive delivery controller for Kubernetes — plus the
> `kubectl argo rollouts` plugin.** Replaces the built-in
> `Deployment` rollout with a `Rollout` CRD that supports
> blue/green, canary, traffic-shifted canary (Istio / Gateway
> API / SMI / Traefik / NGINX / ALB), and analysis-driven
> automatic promotion or abort. Pinned to **v1.9.0**
> (per upstream releases page, verified 2026-05-01)
> ([LICENSE](https://github.com/argoproj/argo-rollouts/blob/master/LICENSE),
> Apache-2.0).

Source: <https://github.com/argoproj/argo-rollouts>

## TL;DR

A `Rollout` looks almost identical to a `Deployment` (same
`spec.template`, same `spec.selector`) but adds a `strategy`
that lists explicit steps — `setWeight: 25` → `pause: {duration: 5m}`
→ `setWeight: 50` → `analysis: { templates: [...] }` → `setWeight: 100`.
The controller reconciles ReplicaSets to honour the weight
schedule and (when wired to a service mesh / ingress) updates
the underlying traffic-split object so the percentage is real
client traffic, not just pod count. `AnalysisTemplate` /
`AnalysisRun` CRDs query Prometheus / Datadog / New Relic / a
web hook and auto-abort if SLOs slip. The `kubectl argo rollouts`
plugin gives `get rollout`, `pause`, `promote`, `abort`,
`set image` against a live cluster plus a local dashboard
(`kubectl argo rollouts dashboard`) that visualises the step
graph and live metrics.

## Install

```bash
# Controller (CRDs + Deployment, install-time only)
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts \
  -f https://github.com/argoproj/argo-rollouts/releases/download/v1.9.0/install.yaml

# kubectl plugin (Homebrew on macOS / Linux)
brew install argoproj/tap/kubectl-argo-rollouts

# kubectl plugin (manual)
curl -LO https://github.com/argoproj/argo-rollouts/releases/download/v1.9.0/kubectl-argo-rollouts-linux-amd64
chmod +x kubectl-argo-rollouts-linux-amd64
sudo mv kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts
```

## Example

```bash
# Inspect a running rollout (steps, current weight, analysis)
kubectl argo rollouts get rollout my-app -n prod --watch

# Trigger a new revision and let the canary schedule run
kubectl argo rollouts set image my-app \
  my-app=ghcr.io/acme/my-app:1.42.0 -n prod

# Manually promote past the next pause step
kubectl argo rollouts promote my-app -n prod

# Abort and roll back to the stable ReplicaSet
kubectl argo rollouts abort my-app -n prod

# Local dashboard for the current kube-context
kubectl argo rollouts dashboard
```

## When to use

- You want canary or blue/green semantics with real traffic
  shifting (mesh / ingress weight), not just pod-count steps,
  and you want the controller to auto-abort on bad metrics.
- You are already on the Argo stack (Argo CD + Argo Workflows)
  and want one CRD model end-to-end.
- You need an audit trail of "step N at time T promoted /
  paused / aborted by user U" — the `Rollout` status carries
  it natively.

## When NOT to use

- A simple `Deployment` rolling update is enough — `Rollout`
  is an extra CRD, an extra controller, and an extra mental
  model. Don't pay the tax for stateless services that don't
  need progressive delivery.
- You want feature flags / per-user targeting — that is
  LaunchDarkly / Unleash / OpenFeature territory, not a
  Kubernetes traffic-shifting controller.
- You are on Flagger already — Flagger is the Flux-side
  equivalent. Pick one progressive-delivery controller per
  cluster and stick with it.
