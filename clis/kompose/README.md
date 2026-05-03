# kompose

- **Repo:** https://github.com/kubernetes/kompose
- **Version:** v1.38.0 (latest stable, 2026-01-15)
- **License:** Apache-2.0 ([LICENSE](https://github.com/kubernetes/kompose/blob/main/LICENSE))
- **Language:** Go
- **Install:** `brew install kompose` · `pacman -S kompose` · static binary releases on the GitHub release page (`kompose-darwin-arm64`, `kompose-linux-amd64`, `kompose-windows-amd64.exe`) · `go install github.com/kubernetes/kompose@latest` · official container `quay.io/kompose/kompose:v1.38.0`

## What it does

`kompose` translates a Docker Compose file (`docker-compose.yml` /
`compose.yaml`) into a set of Kubernetes (or OpenShift) resource
manifests. Run `kompose convert` in a directory containing a Compose
file and it walks the `services:` map, emitting a `*-deployment.yaml`
plus a `*-service.yaml` for each service, a `*-pvc.yaml` for each
named volume, a `*-configmap.yaml` for each `env_file`, a
`*-networkpolicy.yaml` per Compose `networks:` block, and so on. The
mapping is conservative and explicit: `image:` becomes the container
image, `ports:` becomes a `Service` with `type: ClusterIP` (override
to `NodePort` / `LoadBalancer` with `--type` or per-service labels),
`environment:` becomes container `env:` entries, `volumes:` becomes
`PersistentVolumeClaim`s with `ReadWriteOnce` (override to
`ReadWriteMany` with `--volumes`), `depends_on:` becomes nothing on
its own (Compose ordering does not have a Kubernetes equivalent — use
a probe or an init container), and `deploy.replicas:` (Swarm syntax)
becomes the `Deployment.spec.replicas`. Compose-specific labels
(`kompose.service.type`, `kompose.volume.size`,
`kompose.controller.type`, `kompose.image-pull-policy`) override the
defaults per service so a Compose file can be annotated for
production-shaped Kubernetes output without leaving Compose. Output
formats are plain YAML manifests, JSON, a Helm chart skeleton
(`--chart`), or Kustomize bases (`--out kustomize/`). The tool is a
Kubernetes SIG project (lives under `github.com/kubernetes/kompose`,
not in a vendor org) and a graduate of the Cloud Native Sandbox.

## When to pick it / when not to

Pick `kompose` when you have an existing Compose stack — usually
either a dev-loop `docker compose up` for a multi-service app or a
small production deploy on a single Docker host — and the time has
come to put it on a Kubernetes cluster. The first conversion gets you
80% of the way there in 60 seconds: the manifests are correct,
runnable with `kubectl apply -f .`, and shaped the way a human would
hand-write them, so you can then edit them in place rather than treat
the YAML as compiler output. Pair with [`helmfile`](../helmfile/) or
[`kustomize`](../kustomize/) when you need per-environment overlays
that Compose's single-file model does not give you, with
[`kpt`](../kpt/) when you want to keep the converted manifests in a
package-shaped repo that pulls upstream changes, with
[`skaffold`](../skaffold/) for the inner-loop dev experience that
Compose was originally giving you, and with
[`podman`](../podman/) `kube generate` as a sibling tool that does
the same Compose→K8s translation for podman-managed pods.

Skip it as a long-term build step — the recommended pattern is "run
`kompose convert` once, commit the YAML to git, hand-edit from there".
Re-running it on every Compose change overwrites your manual edits
unless you carefully diff. Skip it for Compose stacks that lean on
features Kubernetes does not have a clean equivalent for — `extends:`,
`depends_on:` with healthcheck conditions, host-mounted bind volumes
that are actually devbox source trees, network aliases that double as
DNS hacks; the conversion will warn but the result will need surgery.
Skip it when the destination is not Kubernetes — for a `swarm
deploy`-shaped target, Compose stays as Compose; for a Nomad target,
use Nomad's own `pack` / `job` files; for an ECS target, use AWS
Copilot or `ecs-cli` instead. Skip it for greenfield apps — Compose →
Kubernetes is a migration path, not a design pattern; if you are
starting from scratch and the target is Kubernetes, write Helm or
Kustomize from day one.

## Example invocations

```bash
# In a directory with docker-compose.yml, emit Kubernetes manifests
kompose convert
ls *.yaml
# api-deployment.yaml api-service.yaml db-deployment.yaml db-service.yaml ...

# Convert to a single combined manifest instead of per-service files
kompose convert --out k8s.yaml

# Convert to a Helm chart skeleton (chart name = current dir)
kompose convert --chart
ls templates/

# Convert to OpenShift resources (DeploymentConfig + Route instead of
# Deployment + Ingress)
kompose convert --provider openshift

# Convert and pipe straight to a cluster (one-off; not recommended for prod)
kompose convert --stdout | kubectl apply -f -

# Use a non-default Compose file
kompose -f docker-compose.prod.yml convert --out manifests/

# Annotate a service inside compose.yaml to override the default mapping:
#   services:
#     api:
#       image: ghcr.io/me/api:1.2.3
#       labels:
#         kompose.service.type: LoadBalancer
#         kompose.image-pull-policy: Always
#         kompose.controller.type: statefulset
# Then convert and the api will become a StatefulSet behind a LoadBalancer.

# Inspect the version Kompose was built against
kompose version
```
