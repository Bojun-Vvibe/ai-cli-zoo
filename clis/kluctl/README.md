# kluctl

> **Templated, environment-aware deployment tool for Kubernetes.**
> A single Go binary (and an optional GitOps controller) that takes
> a tree of Kustomize bases + Jinja2-templated YAML + a typed
> "deployment project" config and renders it into per-environment
> manifests, then applies them to a cluster with diff-aware,
> dependency-ordered, prune-safe rollouts. Pinned to **v2.27.0**
> (SPDX: `Apache-2.0`,
> [LICENSE](https://github.com/kluctl/kluctl/blob/main/LICENSE)).

Source: <https://github.com/kluctl/kluctl>

## TL;DR

`kluctl` answers one question — "I have one app, five environments
(dev / staging / prod-eu / prod-us / canary), and I want a single
declarative project that renders the right manifests for each, shows
me a real diff before I apply, and prunes resources I removed —
without writing a Helm chart, without learning Kustomize patches by
heart, and without giving up Jinja2 templating." — with one CLI.

The mental model is three layers stacked deliberately:

1. **Kustomize bases** at the leaves (one per workload).
2. **Jinja2 templates** wrapping the bases for environment-specific
   substitution (image tags, replica counts, hostnames, feature
   flags).
3. **A `.kluctl.yaml` deployment project** at the root, declaring
   targets (= environments), arg schemas, and the dependency order
   between deployments.

Day-to-day commands look like:

```sh
kluctl diff -t prod-eu          # show what would change in prod-eu
kluctl deploy -t prod-eu        # apply with confirmation prompt
kluctl prune -t prod-eu         # delete resources removed from the project
kluctl render -t prod-eu        # dump rendered YAML to disk for inspection
kluctl validate -t prod-eu      # client-side validation only
```

Each command accepts the same `-t <target>` flag, so the same
project source produces 5 environments with no per-environment fork.

## Install

```sh
# macOS via Homebrew
brew install kluctl/tap/kluctl

# Any platform via the install script
curl -sSL https://github.com/kluctl/kluctl/releases/download/v2.27.0/kluctl_v2.27.0_$(uname -s | tr A-Z a-z)_$(uname -m | sed 's/x86_64/amd64/').tar.gz \
  | tar -xz -C /tmp && sudo mv /tmp/kluctl /usr/local/bin/

# Or via Go
go install github.com/kluctl/kluctl/v2/cmd/kluctl@v2.27.0
```

Verify:

```sh
kluctl version
# Version: v2.27.0
```

## License

Apache-2.0 — unrestricted. Safe to embed in CI runner images, ship
inside an internal platform-team distribution, vendor into a
proprietary deployment SDK, or include in a paid managed-Kubernetes
offering.

## Concrete example: one repo, five targets, dependency-ordered apply

`kluctl-project.yaml` at the repo root:

```yaml
targets:
  - name: dev
    context: kind-dev
    args:
      env: dev
      replicas: 1
  - name: staging
    context: aks-staging
    args:
      env: staging
      replicas: 2
  - name: prod-eu
    context: aks-prod-eu
    args:
      env: prod
      replicas: 5
      region: eu
  - name: prod-us
    context: aks-prod-us
    args:
      env: prod
      replicas: 5
      region: us

args:
  - name: env
    default: dev
  - name: replicas
    type: int
  - name: region
    default: ""
```

`deployment.yaml`:

```yaml
deployments:
  - path: namespaces
  - barrier: true               # wait for namespaces to settle
  - path: cert-manager
  - path: external-secrets
  - barrier: true               # secrets must exist before workloads
  - path: workloads/api
  - path: workloads/worker
  - path: workloads/web
```

`workloads/api/deployment.yaml` (Jinja2-templated):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: {{ args.env }}-app
spec:
  replicas: {{ args.replicas }}
  template:
    spec:
      containers:
        - name: api
          image: registry.example.com/api:{{ args.image_tag | default('latest') }}
{% if args.env == 'prod' %}
          resources:
            requests: { cpu: 500m, memory: 512Mi }
            limits:   { cpu: 2000m, memory: 2Gi }
{% endif %}
```

Then:

```sh
kluctl diff -t prod-eu --arg image_tag=v1.42.0
kluctl deploy -t prod-eu --arg image_tag=v1.42.0 --yes
```

The diff shows exactly which resources change (color-coded, three-
way against the live cluster state, not just against the last
apply). The deploy honors `barrier:` markers, so cert-manager fully
rolls out before any workload that depends on a `Certificate` is
applied. Re-running with a removed `path:` entry plus
`kluctl prune` deletes only resources that were managed by the
project — no ambient cluster state is touched.

## Niche

`kluctl` covers the "templated multi-env deploy" gap between three
Kubernetes-native patterns, none of which solve it cleanly for the
"one app, many envs, full diff" use case:

- **Helm** — packages-and-values: great for distributing third-party
  charts, awkward for in-house apps because `values.yaml` becomes a
  10-environment monster and `helm diff` is a third-party plugin
  with limited fidelity.
- **Kustomize alone** — patches and overlays: deterministic and
  template-free, but "bump the image tag from CI" requires either
  `kustomize edit` (mutates files) or a wrapper script. No native
  per-env arg system, no built-in diff tooling.
- **Argo CD / Flux** — GitOps controllers: assume someone else
  produced the manifests; they reconcile the cluster *to* a
  rendered tree. Use kluctl (or kluctl's own GitOps controller) to
  *produce* that tree.

`kluctl` is the in-between layer: Kustomize bases + Jinja2
templating + a typed project model + a real diff/apply/prune CLI.

## Why use it

1. **Per-environment args without a values.yaml mountain.** A typed
   `args:` schema declared once at the project level beats 5 copies
   of an unstructured `values-prod-eu.yaml` whose keys silently
   diverge over time.
2. **Three-way diff against live cluster state.** `kluctl diff`
   compares (rendered manifests, last applied annotation, current
   live spec) — so it correctly identifies "this field was set by a
   webhook / by another controller / by hand" and refuses to silently
   overwrite. Helm and raw `kubectl apply` cannot do this.
3. **Dependency ordering and barriers.** `barrier: true` and
   `waitReadiness: true` markers in `deployment.yaml` let the
   project model "namespaces before secrets before workloads"
   without a wrapper script. Failed barriers abort cleanly with no
   half-applied state.
4. **Prune that you can trust.** Resources are tagged with a project
   ID; `kluctl prune` deletes only what the project once managed
   and no longer does. No `kubectl delete -f old.yaml`
   choreography.
5. **Optional GitOps controller, optional CLI-only.** The same
   project source can be driven from a developer laptop (`kluctl
   deploy`) *or* by an in-cluster `KluctlDeployment` CR that the
   project's own controller reconciles. Same renderer, same diff
   semantics, no behavioral drift between "I ran it" and "the
   controller ran it."
6. **Sealed-secrets / SOPS / ExternalSecrets integration.**
   Encrypted-at-rest secret patterns slot into the templating layer
   without bespoke wrappers.

## Vs already cataloged

- **vs [`helm`](../helm/)** — Helm is a package format + templating
  language + chart registry. Use Helm for distributing reusable
  third-party charts; use kluctl for in-house apps with many
  environments. The two compose: kluctl can render and deploy Helm
  charts via its `helmChart:` deployment item.
- **vs [`kustomize`](../kustomize/)** — Kustomize is the patch /
  overlay engine. kluctl uses Kustomize internally as the leaf-level
  "render this base" primitive and adds the templating, project
  model, diff, prune, and orchestration layer above it.
- **vs [`flux`](../flux/) / [`argocd`](../argocd/)** — those are
  GitOps reconcilers. kluctl provides both a CLI *and* its own
  GitOps controller; if you already run Argo or Flux, kluctl can
  generate the manifests they sync. If you want a single tool for
  rendering + applying + reconciling, kluctl's controller covers
  it.
- **vs [`kpt`](../kpt/) / [`cdk8s`](https://github.com/cdk8s-team/cdk8s)
  / [`pulumi`](../pulumi/)** — kpt is package-and-function-pipeline
  (similar mental model, weaker templating); cdk8s/Pulumi are
  general-purpose-language-as-config (more power, more rope). kluctl
  sits in the YAML-plus-Jinja2 middle ground that most platform
  teams actually want for app deployments.
- **vs [`skaffold`](../skaffold/) / [`tilt`](../tilt/)** — those
  optimize the inner dev loop ("build image, hot-reload into a dev
  cluster"). kluctl optimizes the outer deploy loop (envs,
  promotion, drift detection). They compose: tilt for `kind dev`,
  kluctl for everything from staging onward.

## Caveats

- **Jinja2 templating is a footgun on YAML.** Indentation-sensitive
  output language plus a templating engine that doesn't know about
  YAML indentation produces subtle whitespace bugs. Mitigations:
  prefer Kustomize patches over Jinja2 for structural changes;
  reserve Jinja2 for scalar substitutions (image tags, replica
  counts, env-flag bools); always run `kluctl render` and inspect
  the output during development.
- **The deployment project model has a learning curve.** The
  `.kluctl.yaml` + `deployment.yaml` + per-target args schema
  combination takes a half-day to internalize. Once internalized,
  it removes a lot of friction; before then, plain Kustomize feels
  simpler.
- **Cluster-context handling is per-target.** `targets:` reference
  kubeconfig contexts by name; CI environments need a deliberate
  pattern for materializing those contexts (typically a script that
  writes a synthesized kubeconfig from secrets). Not unique to
  kluctl, but worth planning for before adoption.
- **Prune relies on a project-ID label.** Resources created outside
  kluctl, or by a previous kluctl project under a different ID, are
  invisible to `kluctl prune`. The label model is documented and
  stable, but migrating an existing cluster *into* kluctl
  management requires a one-time `kluctl deploy --replace-on-error`
  pass to re-tag.
- **Jinja2 + Go templates do not coexist.** kluctl uses Jinja2
  exclusively, so existing Helm charts cannot be templated through
  kluctl's renderer (use the `helmChart:` deployment item instead,
  which invokes Helm's own renderer and pipes the output through
  kluctl's apply layer).
- **Last release v2.27.0 is July 2025.** Repository remains active
  on `main` with regular merges; the project's pace reflects API
  stability rather than abandonment. Pin v2.27.0 and re-evaluate
  at the next minor.

## How `kluctl` fits the LLM-CLI workflow

- **Agent-generated env promotions:** an agent that needs to "bump
  image tag in staging from v1.4.2 to v1.4.3" produces a one-line
  arg change rather than editing five `values.yaml` files. The
  diff produced by `kluctl diff -t staging --arg image_tag=v1.4.3`
  is exactly the manifest change that will land — readable enough
  to feed back into the agent for self-review.
- **Pre-merge safety gate:** wire `kluctl diff -t prod-eu` into a
  CI step that posts the diff as a PR comment (via
  [`reviewdog`](../reviewdog/) or a one-off script) so reviewers
  see exactly which production resources change before approval.
- **Drift detection:** scheduled `kluctl diff -t prod-*` runs that
  alert on non-zero diff catch out-of-band cluster mutations
  (manual `kubectl edit`, runaway controllers) before they
  silently change the next deploy's behavior.
- **Disaster-recovery rehearsal:** `kluctl render` produces the
  full set of manifests for an environment as plain YAML, suitable
  for offline backup, audit log, or "rebuild this cluster from
  scratch" runbooks.
