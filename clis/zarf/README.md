# zarf

- **Repo:** https://github.com/zarf-dev/zarf
- **Version:** v0.75.1 (latest stable)
- **License:** Apache-2.0 ([LICENSE](https://github.com/zarf-dev/zarf/blob/main/LICENSE))
- **Language:** Go
- **Install:** `brew install zarf` · `asdf plugin add zarf && asdf install zarf latest` · static binaries on the GitHub release page (`zarf_v0.75.1_Darwin_arm64`, `zarf_v0.75.1_Linux_amd64`, `zarf_v0.75.1_Windows_amd64.exe`) · official container `ghcr.io/zarf-dev/zarf/zarf:v0.75.1`

## What it does

`zarf` is a declarative packager + installer for Kubernetes workloads
that targets the **air-gapped / disconnected / DMZ-network** case as
its primary deployment surface. The model is two phases. **Phase one
("create"), on a connected build host:** you write a `zarf.yaml`
declaring `components` (each a bundle of Helm charts + raw manifests +
container images + Git repositories + arbitrary OCI artifacts + shell
actions), then run `zarf package create .`. Zarf walks every
component, pulls every container image referenced by every chart and
every manifest (it parses them with the same image-resolution logic
your cluster will use), pulls every Git repo at the requested ref,
pulls every Helm chart and renders it to discover transitive image
references, and writes the whole thing out as a single signed,
content-addressed `zarf-package-*.tar.zst` tarball — typically a few
hundred MB to a few GB, fully self-contained, no further internet
egress required to install. **Phase two ("deploy"), on the
target air-gapped cluster:** you `scp` (or sneakernet on a USB) the
tarball over, run `zarf init` once to install Zarf's in-cluster
support stack (a Docker registry, a Gitea Git server, an optional
agent webhook that mutates pod specs to point at the in-cluster
registry instead of `docker.io`), then `zarf package deploy
zarf-package-myapp.tar.zst`. Zarf pushes every bundled image into the
in-cluster registry, every bundled Git repo into the in-cluster
Gitea, applies every chart and manifest in order, and runs every
`actions:` shell hook. The same package, byte-for-byte, deploys to a
laptop k3d cluster, a corporate dev cluster, a customer's regulated
on-prem cluster, and a literally-airgapped lab — the only thing that
changes is which cluster `kubectl` points at. There is no separate
"online mode" vs "offline mode"; the package is the unit of
distribution, always. Zarf is a CNCF Sandbox project as of 2024.

## When to pick it / when not to

Pick `zarf` whenever the cluster you ship to does **not** have
reliable, unrestricted egress to `docker.io`, `quay.io`,
`ghcr.io`, public Helm repos, and GitHub.com — i.e. anything in a
classified network, an OT / industrial environment, a regulated
healthcare / finance environment, an edge site over satellite, a
ship, a forward-deployed military environment, a customer site
behind a corporate proxy that allows-lists nothing useful, or a
"day-one bringup" cluster on hardware that has not yet been let
through the firewall. Zarf inverts the normal Kubernetes assumption
that the cluster pulls from the internet during install: with Zarf,
the connected build host pulls everything once into a tarball, and
the cluster pulls only from itself. Pick it also for repeatable
demos and conference workshops — `zarf package deploy
my-demo.tar.zst` against a fresh `k3d cluster create` is a 60-second
cold start with zero internet, no flaky `image pull` failures during
your talk. Skip Zarf for a normal cloud cluster with full egress and
a real CI/CD pipeline (use Helm + ArgoCD/Flux directly), for
single-binary edge agents that are not Kubernetes (use
[`flux`](../flux/) or vanilla `kubectl apply` from a CI runner), and
for cases where the operator does not want to install Zarf's
in-cluster registry + Gitea support stack (it is a real footprint:
a few hundred MB of pods running permanently, plus the storage for
the registry). The biggest mental adjustment: every container image
in your cluster, after `zarf init`, is referenced by its
in-cluster-registry URL, not its original public URL — Zarf's
mutating webhook handles that rewrite transparently for pods deployed
through Zarf, but you have to know it is happening when you debug.

## Vs already cataloged

- **Vs [`helm`](../helm/) / [`helmfile`](../helmfile/):**
  complementary, not alternative. A Zarf component typically *is* a
  Helm chart (or a list of them); Zarf packages the chart together
  with every image it references, then `helm install`s it on deploy.
  If your network has full egress, Helm + Helmfile is enough; the
  moment it does not, Zarf wraps the same chart and makes it
  shippable across an air gap.
- **Vs [`flux`](../flux/) / [`argocd`](../argocd/):** different
  layer. Flux/Argo are continuous reconcilers that pull manifests
  from a Git repo the cluster can reach. They assume connectivity.
  Zarf is the *bootstrap and supply* layer — once Zarf has installed
  Gitea in-cluster and pushed your manifests there, you can point
  Flux/Argo at that in-cluster Gitea and reconcile from it, fully
  offline.
- **Vs [`kustomize`](../kustomize/) / [`kpt`](../kpt/) /
  [`kompose`](../kompose/):** orthogonal. Those are manifest-shape
  tools (overlay, package, convert). Zarf consumes their output as
  raw `manifests:` inside a component.
- **Vs [`apko`](../apko/) / [`ko`](../ko/) / [`pack`](../pack/):**
  different scope. Those build a single container image. Zarf
  bundles the cluster-shaped *collection* of images plus the
  manifests that wire them together.
- **Vs [`skopeo`](../skopeo/):** Zarf includes skopeo-shaped
  image-copying as one tool inside a much larger workflow. If all
  you need is "copy these N images from public registries to a
  private one", `skopeo sync` is the right tool. If you need
  "copy these N images **plus** their manifests **plus** their Helm
  charts **plus** their CRDs **plus** seed Git repos and run install
  hooks, all from one tarball", that is Zarf.

## Caveats

- **Zarf installs an opinionated in-cluster support stack** (Docker
  registry, Gitea, mutating webhook agent). On clusters where you
  already run a private registry (Harbor, ECR pull-through cache,
  Artifactory) you can point Zarf at it via `zarf init --registry-url
  ...` instead, but the default install adds those pods. Plan
  storage for them; the in-cluster registry will hold every image
  for every package you deploy.
- **Image references in deployed manifests are rewritten** by Zarf's
  mutating admission webhook to point at the in-cluster registry.
  This is the magic that makes packages portable, but it means
  `kubectl describe pod` shows an unexpected URL. Document this for
  the operations team before the first 2 a.m. page.
- **Package size grows quickly.** A platform-engineering bundle with
  Istio + Prometheus stack + Loki + Grafana + a couple of apps is
  routinely 5–15 GB. Zstd-compressed
  ([see `zstd`](../zstd/)) inside the tarball, but still: budget
  storage on the build host and the sneakernet medium.
- **Signed packages need cosign keys you manage.** `zarf package
  create --signing-key ...` and `zarf package deploy --key ...`
  enforce signatures; without keys, deployment falls back to a
  warning. For regulated targets, set up the cosign key pipeline
  first ([`cosign`](../cosign/) is already in the catalog).
- **Project history note:** Zarf began life inside Defense Unicorns
  and moved to the vendor-neutral `zarf-dev` GitHub org when it
  joined the CNCF Sandbox in 2024. Older docs and blog posts may
  reference the old `defenseunicorns/zarf` import path; the
  repo redirects, but pin the new path in any tooling.
- Apache-2.0 ([LICENSE](https://github.com/zarf-dev/zarf/blob/main/LICENSE))
  — permissive; safe to ship the `zarf` binary inside customer-
  delivered installers and to embed Zarf packages inside commercial
  air-gapped distributions.
