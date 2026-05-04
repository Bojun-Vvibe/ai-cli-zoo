# istioctl

> **The control-plane CLI for Istio service mesh.** A single Go
> binary that installs / upgrades / uninstalls Istio in a
> Kubernetes cluster, validates and diffs the mesh
> configuration (`VirtualService`, `DestinationRule`,
> `Gateway`, `AuthorizationPolicy`, `PeerAuthentication`,
> `Telemetry`, …), and inspects the Envoy sidecar state of
> any pod in the mesh. Pinned to **v1.29.2** (released
> 2026-04-13, SPDX: `Apache-2.0`,
> [LICENSE](https://github.com/istio/istio/blob/master/LICENSE)).

Source: <https://github.com/istio/istio> (`istioctl/`)

## Repo

- URL: <https://github.com/istio/istio>
- Subpath: `istioctl/`
- Owner/org: istio (CNCF graduated project, 2025)
- License file: [LICENSE](https://github.com/istio/istio/blob/master/LICENSE)

## Version

`v1.29.2` — released 2026-04-13. Verify with `istioctl version`.
Each minor (`1.28`, `1.29`, `1.30`) is supported by the project
for ~6 months from release; pin the CLI to the same minor as
the in-cluster control plane (`istiod`) so install / upgrade
paths stay supported. Distributed as
`istioctl-1.29.2-<os>-<arch>.tar.gz` (~25–30 MB) from the
release page, and as part of the larger `istio-1.29.2-<os>.tar.gz`
that also contains the sample manifests and addons.

## License

**Apache-2.0** — OSI-approved, permissive. Safe to embed in CI
runner images, ship inside an internal platform-engineering
bootstrap, or vendor into a customer-shipped Kubernetes
distribution. Istio became a CNCF graduated project in 2025;
the OSS posture is stable.

## What it does

The four primary jobs:

1. **Install / upgrade / uninstall the control plane.**
   - `istioctl install --set profile=demo|default|minimal|ambient|empty`
     applies the bundled `IstioOperator` profile, rendering
     all `istiod` / ingress / egress / CRD resources to the
     cluster.
   - `istioctl upgrade` performs a canary control-plane
     upgrade with revision tags so workloads can migrate to
     the new revision pod-by-pod.
   - `istioctl uninstall --purge` removes everything cleanly,
     including CRDs.
   - The recommended flow today is **revision-based** (each
     install gets a `revision=` tag, sidecars opt in by
     namespace label, both control planes coexist during
     migration).
2. **Generate / diff / validate manifests offline.**
   - `istioctl manifest generate` renders the operator config
     to raw YAML for GitOps pipelines (apply via Argo CD,
     Flux, etc.).
   - `istioctl manifest diff` compares two rendered manifests.
   - `istioctl validate -f mesh-config.yaml` validates user
     CRDs (`VirtualService`, `Gateway`, `AuthorizationPolicy`,
     …) against the schema *and* against semantic rules
     (route precedence, host conflicts, port mismatches).
3. **Analyze a live mesh for misconfiguration.**
   - `istioctl analyze` runs ~60 built-in checks against the
     in-cluster config (sidecar-injection mismatches,
     unreachable services, conflicting `VirtualService`
     hosts, `mTLS` mode mismatches between
     `PeerAuthentication` and `DestinationRule`, missing
     `Gateway` selectors, deprecated APIs). The single most
     useful command for a "the mesh is broken, where do I
     start" debugging session.
4. **Inspect the Envoy sidecar of any mesh pod.**
   - `istioctl proxy-status` — control-plane sync status of
     every sidecar (CDS / LDS / EDS / RDS), shows which
     proxies have stale config.
   - `istioctl proxy-config cluster|listener|route|endpoint|
     bootstrap|secret <pod>` — dumps the live Envoy config
     of one sidecar (the canonical "what does this proxy
     actually believe right now" view).
   - `istioctl x describe pod <pod>` — synthesizes mesh
     policy, traffic routing, and mTLS posture for one pod
     into a human-readable report.
   - `istioctl pc log <pod> --level=debug` — change Envoy log
     level on a live sidecar.

`istioctl experimental ambient` (and the `ambient` install
profile) operate the **ambient mesh** (no per-pod sidecar,
ztunnel + waypoint proxies) introduced in 1.18+ and progressed
to GA in the 1.22+ line.

## When to use

- **You install or upgrade Istio.** The supported install
  surface is `istioctl install` or the operator manifests it
  generates. Helm chart and operator are also supported, but
  istioctl is the canonical entry point.
- **You ship mesh CRDs in git.** `istioctl validate` in CI
  catches schema and semantic errors (host overlap, port
  mismatch, conflicting policies) before they reach the
  cluster — significantly lower MTTR than discovering them
  via 503s.
- **You debug a misbehaving sidecar.** `proxy-status` +
  `proxy-config` + `analyze` are the canonical triage
  triplet. `kubectl exec` into the pilot-agent will give
  you raw Envoy admin, but istioctl wraps the most common
  questions.
- **You operate a multi-revision migration.** Canary
  upgrades with revision tags are the supported path off
  an old control plane onto a new one without touching
  workloads first.

## When NOT to use

- **You haven't decided whether you need a service mesh.**
  Istio is heavy: per-pod sidecars (or per-node ztunnels in
  ambient), an extra control plane, an extra CA, complex
  CRDs. If your only goal is mTLS between services or
  basic L7 routing, evaluate
  [`linkerd`](../linkerd/) (smaller surface, opinionated)
  or a Gateway-API-only setup first.
- **You only need ingress.** Pure ingress is what
  [`traefik`](../traefik/) and the cloud-native
  Gateway-API implementations target. Istio's
  ingress-gateway is part of a mesh; do not adopt the
  whole mesh just for ingress.
- **You are on a managed mesh** (GKE Anthos Service Mesh,
  AWS App Mesh, OCI Service Mesh). The vendor's CLI is
  the supported surface there; istioctl may still work
  but is not the first-line tool.

## Alternatives in this catalog

- [`linkerd`](../linkerd/) — Rust data plane, smaller
  surface area, opinionated defaults. Pick `linkerd` when
  the team prefers fewer knobs and the workload is mostly
  HTTP/gRPC; pick `istioctl` when you need the broader
  feature set (request hedging, rich AuthorizationPolicy,
  ambient mesh, multi-cluster mesh, EnvoyFilter escape
  hatch).
- [`cilium-cli`](../cilium-cli/) — Cilium's CLI for the
  eBPF-backed CNI and *Cilium Service Mesh*. Pick when the
  CNI and mesh are unified into one component; pick
  `istioctl` for the broader Istio ecosystem (extensions,
  Wasm filters, multi-cluster topologies that predate
  Cilium ClusterMesh).
- [`kubeshark`](../kubeshark/) — packet-level traffic
  observer; complementary to istioctl for L4 / L7 forensic
  inspection. Use both when "what did this sidecar
  actually see" is the question.
- [`k9s`](../k9s/) — TUI for Kubernetes resources; useful
  for browsing the mesh CRDs but does not validate them.
- [`grpcurl`](../grpcurl/) / [`grpcui`](../grpcui/) —
  hit a meshed gRPC service through the ingress gateway
  to verify routing behaves as the `VirtualService` says.

## AI-native angle

istioctl is not an LLM tool, but its structured outputs sit
naturally inside an agent verification loop:

- **Deterministic mesh-config validation.** When an agent
  ([`aider`](../aider/), [`opencode`](../opencode/),
  [`claude-code`](../claude-code/), [`codex`](../codex/))
  edits a `VirtualService` / `DestinationRule` /
  `AuthorizationPolicy`, `istioctl validate -f` returns a
  structured pass/fail with the offending field and reason
  — a stronger feedback signal than the LLM re-reading its
  own YAML.
- **`istioctl analyze --output-format json`** emits machine-
  readable diagnostics (`code`, `level`, `message`,
  `origin`) that an agent can parse, prioritize, and
  reason about before approving a PR that touches mesh
  config.
- **Manifest diff as a plan step.** `istioctl manifest
  generate` rendered before/after a config change is a
  deterministic diff an agent can present in the PR
  description ("this change adds 1 listener and 3 routes
  to the ingress gateway") — the mesh equivalent of
  `terraform plan`.
- **Proxy-config dumps for grounded debugging.**
  `istioctl proxy-config cluster <pod> -o json` is the
  ground-truth view of what a sidecar currently believes.
  Agents debugging "why is service A getting 503 from B"
  should fetch that, not guess from the user's prose.

## Caveats

- **Version skew is real.** istioctl supports control
  planes within ±1 minor. Upgrade istioctl with the
  cluster, not separately.
- **Ambient is newer.** Ambient mesh is GA but adoption is
  still maturing; some Envoy-filter / WasmPlugin patterns
  from the sidecar world do not yet have ambient
  equivalents. Read the ambient compatibility matrix
  before migrating production.
- **`istioctl install` writes cluster-scoped CRDs.** The
  same CRDs back the in-cluster operator and any GitOps
  apply path; running install + GitOps without revision
  alignment causes drift. Pick one source of truth.
- **`AuthorizationPolicy` defaults are deny-by-default
  *only* when at least one policy selects the workload.**
  An empty policy set means allow-all — a footgun
  `istioctl analyze` will flag with `IST0118` /
  `IST0117`.
- **Sidecar sizing is not free.** A naive `istio-proxy`
  ask for 100m CPU / 128Mi memory per pod adds up across
  thousands of pods. Tune `resources.requests` in the
  `IstioOperator` and use [`pluto`](../pluto/) to spot
  deprecated CRDs before upgrades.

## Concrete example

```sh
# Install Istio 1.29.2 with the default profile + a revision tag.
istioctl install --set profile=default --set revision=1-29-2 -y

# Mark a namespace for sidecar injection on this revision.
kubectl label ns app istio.io/rev=1-29-2 --overwrite
kubectl rollout restart deploy -n app

# Validate user mesh config in CI before apply.
istioctl validate -f mesh/virtualservice.yaml \
                  -f mesh/destinationrule.yaml \
                  -f mesh/authpolicy.yaml

# Analyze the live mesh and emit JSON for an agent / dashboard.
istioctl analyze --all-namespaces -o json > mesh-issues.json

# Triage one pod end-to-end.
istioctl x describe pod -n app web-7d9f9-abcde
istioctl proxy-status
istioctl proxy-config route -n app web-7d9f9-abcde
```
