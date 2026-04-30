# skaffold

- **Repo:** https://github.com/GoogleContainerTools/skaffold
- **Version:** v2.19.0
- **License:** [LICENSE](https://github.com/GoogleContainerTools/skaffold/blob/main/LICENSE) (Apache-2.0)
- **Category:** Inner-loop build/deploy orchestrator for Kubernetes

## What it is

`skaffold` is a CLI that watches your source tree, rebuilds container images
on change, pushes them (or side-loads them into a local cluster), and reapplies
the Kubernetes manifests / Helm charts / Kustomize overlays that reference
those images — all driven by a single `skaffold.yaml`. It is not a CI system
and not a GitOps controller; it is the **edit-save-see-it-running** loop
specifically for "my code lives in a container that runs in a pod." It plugs
into local clusters (kind, k3d, minikube, Docker Desktop) as readily as into
remote dev clusters, and its `dev` mode handles port-forwarding, log tailing,
and image-tag rewriting so you do not hand-edit manifests between iterations.

## Install

```
brew install skaffold                                                  # macOS / Linuxbrew
# or grab the binary from https://github.com/GoogleContainerTools/skaffold/releases
skaffold version
```

## Basic usage

```
skaffold init                          # scaffold skaffold.yaml from existing manifests
skaffold dev                           # watch + rebuild + redeploy + tail logs
skaffold run --tail                    # one-shot build+deploy (CI-friendly)
skaffold debug                         # same as dev, but wires up language debuggers
skaffold render -o rendered.yaml       # produce final manifests without applying
skaffold delete                        # tear down what `run`/`dev` deployed
```

## When to use it

- You are **iterating on a service that only makes sense inside a cluster**
  (sidecars, service mesh, real ingress, real secrets) and `docker run`
  locally is not a faithful environment.
- You want **one tool that knows about Docker/Buildpacks/Bazel/Jib/ko on the
  build side and kubectl/Helm/Kustomize on the deploy side**, with file
  watching wired through.
- You need **the same `skaffold.yaml` to drive both `dev` loops and CI
  builds**, so there is one source of truth for "how this service is built
  and deployed."
- Skip it for **pure GitOps production rollouts** (use Argo CD or Flux), and
  skip it when your service runs fine as a plain `docker compose` stack —
  Skaffold's value is the Kubernetes-native dev loop, not generic container
  orchestration.
