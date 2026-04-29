# helmfile

> **Declarative spec for an entire Helm release set — all your
> charts, their values, their order, and their target clusters
> in one versioned YAML, applied as one transactional `helmfile
> apply` instead of N hand-coordinated `helm upgrade` calls.**
> Pinned to **v1.4.4**,
> [LICENSE](https://github.com/helmfile/helmfile/blob/main/LICENSE),
> MIT.

Source: <https://github.com/helmfile/helmfile>

## TL;DR

`helmfile` is what you reach for when "deploy the platform"
means "install / upgrade 12 Helm charts in a specific order
across 3 namespaces with environment-specific values, then
diff against the live cluster before any apply." It is a
single Go binary that reads `helmfile.yaml` (one file
declaring repositories, releases, value layering, and
cross-release dependencies), and `helmfile apply` runs the
entire reconciliation as one transaction with a built-in
`helm diff` preview gate. Same artifact runs locally and in
CI, environments are first-class (`helmfile -e prod apply`),
and the Helmfile project is a CNCF Sandbox member with active
multi-contributor maintenance after forking from the
unmaintained roboll/helmfile.

## Install

```bash
# Homebrew (macOS / Linux)
brew install helmfile

# Pre-built binary (any platform)
curl -L "https://github.com/helmfile/helmfile/releases/download/v1.4.4/helmfile_1.4.4_$(uname -s | tr A-Z a-z)_amd64.tar.gz" \
  | tar xz helmfile && sudo mv helmfile /usr/local/bin/

# Go install
go install github.com/helmfile/helmfile@v1.4.4

# Required helper plugin (helmfile shells out to it for diff/apply gating)
helm plugin install https://github.com/databus23/helm-diff

# verify
helmfile version          # 1.4.4
helmfile init             # interactive: installs helm-diff, helm-secrets, helm-git if missing
```

## License

MIT — see
[LICENSE](https://github.com/helmfile/helmfile/blob/main/LICENSE).
Permissive: ship the binary in CI images and base images
freely; your `helmfile.yaml` and values stay under your
repo's own license.

## One Concrete Example

A platform repo that installs ingress + cert-manager +
external-dns + monitoring + the app, ordered, per-env:

```yaml
# helmfile.yaml
repositories:
  - name: ingress-nginx
    url: https://kubernetes.github.io/ingress-nginx
  - name: jetstack
    url: https://charts.jetstack.io
  - name: external-dns
    url: https://kubernetes-sigs.github.io/external-dns
  - name: prometheus-community
    url: https://prometheus-community.github.io/helm-charts

environments:
  staging:
    values:
      - environments/staging.yaml
  prod:
    values:
      - environments/prod.yaml

releases:
  - name: cert-manager
    namespace: cert-manager
    chart: jetstack/cert-manager
    version: v1.16.2
    set:
      - name: crds.enabled
        value: true
    values:
      - values/cert-manager/{{ .Environment.Name }}.yaml

  - name: ingress-nginx
    namespace: ingress-nginx
    chart: ingress-nginx/ingress-nginx
    version: 4.11.3
    needs: [cert-manager/cert-manager]
    values:
      - values/ingress-nginx/{{ .Environment.Name }}.yaml

  - name: external-dns
    namespace: external-dns
    chart: external-dns/external-dns
    version: 1.15.0
    needs: [ingress-nginx/ingress-nginx]
    values:
      - values/external-dns/{{ .Environment.Name }}.yaml

  - name: kube-prometheus-stack
    namespace: monitoring
    chart: prometheus-community/kube-prometheus-stack
    version: 67.0.0
    values:
      - values/prometheus/{{ .Environment.Name }}.yaml

  - name: my-app
    namespace: app
    chart: ./charts/my-app          # local chart in this repo
    needs:
      - ingress-nginx/ingress-nginx
      - cert-manager/cert-manager
    values:
      - values/my-app/{{ .Environment.Name }}.yaml
```

Workflow:

```bash
# Show what would change in staging vs the live cluster, no writes
helmfile -e staging diff

# Apply staging (runs diff first, prompts unless --skip-diff-on-install)
helmfile -e staging apply

# Render the entire fleet as plain manifests (for argocd / kustomize input)
helmfile -e prod template > rendered-prod.yaml

# Just one release
helmfile -e prod -l name=my-app apply

# Sync (no diff gate, idempotent upgrade-or-install everything)
helmfile -e prod sync
```

## Niche It Fills

**Multi-release Helm orchestration as one declarative
artifact.** Plain `helm` solves "deploy *one* chart"; once
the platform involves 5+ charts with ordering constraints
(cert-manager CRDs before ingress that uses them, secrets
before the app that mounts them) and per-environment value
overlays, teams either write a brittle bash wrapper around
`helm upgrade --install` or move to a heavyweight GitOps
controller (ArgoCD / Flux) that owns the cluster. Helmfile
sits in the gap: keep using `helm` semantics, but declare
the *whole fleet* (release set, dependency graph, value
layering, environments) in one file and apply / diff / sync
it transactionally. CI runs the same `helmfile -e prod
apply` a developer ran locally against `kind`.

## Why use it

Three things `helmfile` does that plain `helm` / a wrapper
script / kustomize do not, all at once:

1. **Built-in `diff` gate before any write.** `helmfile diff`
   shells out to `helm-diff` for every release and prints
   exactly the manifest delta against the live cluster. CI
   can post the diff on the PR (`helmfile -e prod diff
   --output simple`) so reviewers see "this PR changes the
   ingress controller's `controller.replicaCount` from 2 to
   4 and adds a NetworkPolicy" without reading values files.
2. **Cross-release `needs:` dependency graph.** Declares "X
   depends on Y" as data; `apply` topologically sorts and
   parallelises siblings. The cert-manager-CRDs-before-the-
   `Certificate`-resource race that bites every fresh
   cluster is just `needs: [cert-manager/cert-manager]` in
   YAML, not a bash sleep loop.
3. **First-class environments + Go template + secrets.**
   `environments:` block + `{{ .Environment.Name }}` /
   `{{ .Values.foo }}` template injection lets one
   `helmfile.yaml` cover staging / prod / dev with
   per-env values files and per-env secrets (via
   `helm-secrets` + SOPS). The same artifact serves
   "spin up a dev cluster" and "deploy production"
   without copy-paste.

For a platform repo that wants `helm` semantics, `git`-
reviewable changes, and one command to apply the whole
stack, `helmfile` is the smallest abstraction that does
the job — no controller to install, no CRDs to manage.

## Vs Already Cataloged

- **Vs [`kustomize`](../kustomize/):** different problem.
  `kustomize` is for overlaying / patching plain
  Kubernetes manifests; it does not understand Helm
  charts as templated units. `helmfile` orchestrates
  *Helm releases* (charts + values + lifecycle); the two
  compose — `helmfile` can render a chart and pipe to
  `kustomize`, or call out to charts that include
  `kustomize` patches.
- **Vs [`tilt`](../tilt/) (this round):** different
  audience. Tilt is for the inner dev loop (watch files
  → rebuild → re-apply, fast feedback on a local
  cluster). Helmfile is for the platform / release
  layer: one declarative file describing what should
  exist in a target cluster. Use Tilt during `make
  dev`; use Helmfile in CI and during cluster
  bootstrap.
- **Vs [`stern`](../stern/):** unrelated; `stern` tails
  multi-pod logs. Listed only to clarify they are
  complementary cluster tooling, not competing.
- **Vs ArgoCD / Flux:** different operational model.
  ArgoCD / Flux are *controllers* running in the
  cluster that pull from git continuously. Helmfile is
  a *push* tool you run from CI / locally. Many teams
  use Helmfile for cluster bootstrap (the controllers
  themselves) then ArgoCD for application releases.

## Caveats

- **Helmfile v1.x removed `bases:` and several v0
  compatibility shims.** A repo authored against v0 may
  need migration; the v1 release notes
  (https://github.com/helmfile/helmfile/releases/tag/v1.0.0)
  enumerate every breakage. Pin `helmfile` exactly in CI
  (`helmfile_1.4.4_linux_amd64`) and bump intentionally
  with a `helmfile diff` regression check.
- **`needs:` is *deployment* order, not health-readiness.**
  Helmfile waits for the prior release's `helm upgrade` to
  return, not for its pods to be Ready. For
  cert-manager-style CRD-then-CR ordering, also enable
  `wait: true` and a sensible `timeout:` per release, or
  the next release will start applying before the CRD is
  registered.
- **Templating is Go-template, not Sprig-everywhere.**
  Helmfile uses Go templates with the Sprig function set,
  but the *chart* values are still rendered by Helm with
  Helm's own template engine. There is one render pass at
  the helmfile layer and another at the helm layer; debug
  with `helmfile template --debug`.
- **`helm diff` plugin is required and must match.** A
  fresh CI image without the plugin will fail at first
  `apply`. `helmfile init` or an explicit `helm plugin
  install https://github.com/databus23/helm-diff
  --version v3.10.0` in your image build is the fix; pin
  the plugin version too.
- **Don't bypass `diff` in production.** It is tempting to
  add `--skip-diff-on-install` everywhere to make CI
  green; resist. The diff is the safety net — pipe it to
  the PR comment instead and require a human approve
  step before `apply`.
