# kubevela

> **Application-centric delivery layer on top of Kubernetes** —
> implements the Open Application Model (OAM) so platform teams
> ship a *catalog* of typed components, traits, policies, and
> workflows that app developers consume via one `Application` CR
> without learning the underlying chart / kustomize / operator
> sprawl. The `vela` CLI manages applications, addons (Helm /
> Terraform / Crossplane bridges), and multi-cluster delivery.
> Pinned to **v1.11.0-alpha.3** (released 2026-04-13,
> [`gh api repos/kubevela/kubevela/releases/latest`](https://github.com/kubevela/kubevela/releases/latest),
> [LICENSE](https://github.com/kubevela/kubevela/blob/main/LICENSE),
> Apache-2.0).

Source: <https://github.com/kubevela/kubevela>

## TL;DR

Kubernetes gives you 30+ primitives (Deployment, Service, Ingress,
HPA, NetworkPolicy, PDB, ServiceAccount, ConfigMap, …) and three
templating layers ([`helm`](../helm/), [`kustomize`](../kustomize/),
raw YAML) and an open-ended operator ecosystem. App developers
don't want to learn any of that — they want to say "deploy this
container, expose it on a hostname, autoscale 2-10, roll out
canary-style across `staging` then `prod`." KubeVela is the
abstraction layer that lets a *platform team* define those words
once (as `ComponentDefinition` / `TraitDefinition` /
`PolicyDefinition` / `WorkflowStepDefinition` CUE templates) and
ship them as a catalog the dev team consumes via a single
`Application` CR. Compared to internal Helm-chart-of-charts
patterns: KubeVela's definitions are typed (CUE schema, not Go
templates), the `Application` is a first-class CR with a
controller that reconciles workflow steps + multi-cluster
placement, and the same `vela` CLI does dev-loop (`vela up`),
catalog management (`vela addon enable fluxcd`), and topology
inspection (`vela status`, `vela ls`). It sits one layer *above*
[`argocd`](../argocd/) / [`flux`](../flux/) — you can pair it with
either as the actual GitOps reconciler, while KubeVela owns the
"what does an app look like" model.

## Install

```bash
# Install the vela CLI
curl -fsSl https://kubevela.io/script/install.sh | bash
brew install kubevela           # Homebrew

# Install the controller into the cluster (Helm chart)
helm repo add kubevela https://charts.kubevela.net/core
helm repo update
helm install --create-namespace -n vela-system kubevela kubevela/vela-core

# Or use the CLI's bootstrapper (handles CRDs + addons)
vela install

# verify
vela version
kubectl get pods -n vela-system
```

## Representative examples

```bash
# 1. Minimal Application — one webservice with an ingress trait
#    app.yaml
#    apiVersion: core.oam.dev/v1beta1
#    kind: Application
#    metadata: { name: hello }
#    spec:
#      components:
#        - name: hello
#          type: webservice
#          properties: { image: nginx:1.27, port: 80 }
#          traits:
#            - type: gateway
#              properties: { domain: hello.example.com, http: { "/": 80 } }
#            - type: scaler
#              properties: { replicas: 3 }
vela up -f app.yaml

# 2. Inspect what was reconciled (including multi-cluster topology)
vela status hello
vela ls
vela exec hello -- /bin/sh

# 3. Add an addon (e.g. wire up Flux for Helm-chart components)
vela addon list
vela addon enable fluxcd
vela addon enable terraform-aws

# 4. Define a custom trait (CUE) — "every component gets a sidecar"
#    sidecar.cue
#    "sidecar": {
#      type: "trait"
#      annotations: {}
#      attributes: { appliesToWorkloads: ["deployments.apps"] }
#    }
#    template: { patch: spec: template: spec: containers: [ ... ] }
vela def apply ./sidecar.cue

# 5. Multi-cluster delivery — register a managed cluster, then a
#    policy that places staging in cluster-a and prod in cluster-b
vela cluster join ./cluster-a.kubeconfig --name cluster-a
vela cluster join ./cluster-b.kubeconfig --name cluster-b
#    inside Application:
#    policies:
#      - name: topology-staging
#        type: topology
#        properties: { clusters: [cluster-a], namespace: staging }
#      - name: topology-prod
#        type: topology
#        properties: { clusters: [cluster-b], namespace: prod }
#    workflow:
#      steps:
#        - name: deploy-staging
#          type: deploy
#          properties: { policies: [topology-staging] }
#        - name: manual-approve
#          type: suspend
#        - name: deploy-prod
#          type: deploy
#          properties: { policies: [topology-prod] }

# 6. Dry-run / lint a definition before applying
vela def vet ./sidecar.cue
vela dry-run -f app.yaml
```

## When to use vs. alternatives

- Pick **kubevela** when a platform team owns the developer UX for
  *many* applications and wants a typed, catalog-driven abstraction
  over Kubernetes that survives churn in the underlying primitives
  (replace Deployment with a Knative Service in the
  `webservice` ComponentDefinition once, every consuming
  Application picks it up). Also pick it when multi-cluster
  delivery + workflow steps (suspend, manual-approve, canary,
  parallel) need to be a *first-class* model, not a shell script
  in CI.
- Pick [`argocd`](../argocd/) /  [`flux`](../flux/) when the
  abstraction you actually want is **"sync this Git directory of
  manifests to this cluster"** — they're orthogonal: KubeVela
  describes the *app*, Argo / Flux *delivers* it. Many users run
  KubeVela's Application CR through Argo as the reconciler.
- Pick [`helm`](../helm/) when one team owns one chart for one
  service and the templating ceiling (Go templates, values.yaml)
  is fine. KubeVela's CUE-typed definitions only pay back when
  there's a *catalog* of >5 component shapes shared across teams.
- Pick [`kustomize`](../kustomize/) for pure overlay patching
  without a controller in-cluster — simpler, no CRDs, no runtime.
- Pick [`kpt`](../kpt/) for client-side, function-driven package
  hydration — also CRD-free, also no in-cluster runtime.
- Pick [`skaffold`](../skaffold/) /  [`tilt`](../tilt/) for the
  *inner dev loop* (build → push → deploy on save). KubeVela is
  the *outer loop* (catalog, multi-cluster, governance).
- Caveats: introduces an in-cluster controller and a CRD set you
  now own (upgrades require care — pin the chart, watch
  release notes), CUE has a learning curve for Go-templated-Helm
  veterans, and the alpha/RC tags on recent releases are real —
  this is an active CNCF project but production users typically
  pin to the latest GA minor (`v1.10.x` line) rather than
  `-alpha` builds for prod clusters.
