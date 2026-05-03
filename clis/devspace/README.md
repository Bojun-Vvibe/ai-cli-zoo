# devspace

> **Inner-loop developer CLI for Kubernetes — fast file sync,
> port-forwarding, on-cluster build, and live reload via a single
> declarative `devspace.yaml`.**
> A Go binary that turns a Kubernetes cluster (kind, k3d, a remote
> dev namespace, or a real prod-shaped cluster) into your laptop's
> background runtime: edit code locally, see changes inside the pod
> in seconds, with logs streamed back to your terminal. Pinned to
> **v6.3.21** (released 2026-04-22, SPDX: `Apache-2.0`,
> [LICENSE](https://github.com/devspace-sh/devspace/blob/main/LICENSE)).

Source: <https://github.com/devspace-sh/devspace>

## TL;DR

`devspace dev` reads `devspace.yaml`, swaps the target deployment's
container for a "dev container" image (with your tools and your
source mounted), opens file-sync from your local working directory
into `/app` inside the pod, port-forwards every container port to
localhost, and tails the logs. Edit a file locally → file appears
in the pod → your in-pod hot-reloader (nodemon, `air`, `cargo
watch`, `uvicorn --reload`, …) restarts the process → log line
appears in your terminal. Round-trip on the order of 1–3 seconds
for most stacks.

When you're done, `devspace purge` tears the dev overlay back down
and the original deployment manifest is back in charge.

## Install

```sh
# macOS via Homebrew
brew install devspace

# Linux / WSL via curl
curl -L -o devspace \
  "https://github.com/devspace-sh/devspace/releases/download/v6.3.21/devspace-linux-amd64"
sudo install -c -m 0755 devspace /usr/local/bin

# Verify
devspace version                # 6.3.21
```

## License

Apache-2.0 — unrestricted commercial use with explicit patent
grant. Safe to vendor into onboarding scripts, internal CLI
wrappers, and corporate dev-environment tooling.

## Primary use case

You have a 12-service Kubernetes app. Spinning up the whole graph
on a laptop with Docker Compose is fragile and drifts from prod.
The right place for the app to run is a Kubernetes namespace
(shared dev cluster, or a per-engineer kind/k3d). The wrong place
to *edit* the app is inside the cluster. `devspace dev` bridges
that: prod-shaped cluster, laptop-shaped editor, sub-second edit
loop. The 11 services you're not actively touching keep running
their normal images; the one service you're touching gets a dev
overlay with file sync.

## What it competes with

- **[`tilt`](../tilt/)** — same problem space, different shape.
  Tilt configures the dev loop in a Starlark `Tiltfile` and ships
  with a polished web UI showing every service's status, logs, and
  endpoints. DevSpace configures via YAML (`devspace.yaml`) and
  optionally exposes a UI (`devspace ui`); the YAML-first shape
  composes more naturally with the rest of the Kubernetes config
  ecosystem (Helm, Kustomize). Pick `tilt` for the UI-first
  multi-service dashboard; pick `devspace` for the YAML-first,
  Helm-aware variant.
- **[`skaffold`](../skaffold/)** — Google's "build-deploy-watch"
  inner loop. Strong on the build/deploy primitive (Bazel, Buildah,
  Jib, Kaniko first-class), weaker on the file-sync inner loop
  story. Pick `skaffold` when *build* is the dominant cost; pick
  `devspace` when *file sync + restart* is the dominant cost.
- **`mirrord`** — instead of running your code in the cluster, runs
  it on your laptop and *steals/mirrors* traffic + env + file I/O
  from a target pod into the local process. Different inversion of
  the problem; brilliant for stateful debugging of one service at a
  time. `devspace` keeps your code in the cluster; `mirrord` brings
  the cluster to your code. They are not mutually exclusive.
- **`docker compose up` + a manual `kubectl apply` for staging.**
  The traditional split. Works until your compose file diverges
  from your manifests; then the per-engineer "it works on my Mac
  but not on staging" arc starts. `devspace` keeps you on
  Kubernetes from the start.
- **Telepresence (open source).** Closer to mirrord — proxies
  cluster traffic to a local process. Same trade-off discussion.

## AI-native angle

`devspace` is not an LLM tool, but it's a force multiplier under
agents that touch Kubernetes-shaped services:

- **Sub-3-second feedback for cluster code.** When
  [`aider`](../aider/), [`opencode`](../opencode/),
  [`claude-code`](../claude-code/), or [`codex`](../codex/) edits a
  service, the change is in the cluster pod within ~1 s and the
  in-pod hot-reloader restarts the process within another ~1 s.
  The agent can run an integration test against the live service
  on the next turn. Without `devspace`, the loop is "edit → docker
  build → kubectl rollout → wait for ready → test" — easily 60+ s
  per turn.
- **`devspace.yaml` is a declarative dev contract.** An agent can
  read the YAML to know which container to target, which ports
  matter, which paths get synced, and what env var the in-pod
  process expects — no inference from `Dockerfile` or `Makefile`
  required. This drops the prompt context an agent needs to safely
  modify a service's dev loop.
- **`devspace run-pipeline <name>` exposes named flows.** Pipelines
  are the Tilt/Make-equivalent: `dev`, `deploy`, `purge`, plus
  user-defined ones. Agents can be told "use `devspace
  run-pipeline test` to run the integration suite" without needing
  to know the underlying `kubectl run` / `helm test` mechanics.
- **Hooks at every lifecycle stage.** `before:deploy`, `after:sync`,
  `on-error` hooks let an agent's wrapper inject custom verification
  steps (run linters before sync, capture pod logs on error) into
  the dev loop without forking the YAML.
- **DAG of dependencies.** A `dependencies` block lets a service
  declare "before I can `dev`, run another `devspace.yaml` for the
  shared infra namespace." Agents working on a single service get
  the full dependency graph spun up automatically — no manual
  bootstrapping prompt.

## Caveats

- **The dev container is *not* your prod container.** `devspace
  dev` swaps in an image with `bash`, your toolchain, and your
  source mounted. It is not the production image. Anything you
  test in `dev` mode and not in a separate `deploy` pipeline is
  untested for production. Always run `devspace deploy` (or your
  CI's normal build) before declaring something works.
- **File sync ≠ rebuild.** If your in-pod process needs a compile
  step (Go binary, Rust binary, Java jar), the container's
  hot-reloader (`air`, `cargo watch`, `mvn spring-boot:run`) has to
  do that work. `devspace` syncs the source; the in-container loop
  does the build. Make sure the dev image actually has the
  toolchain.
- **Cluster credentials are real credentials.** `devspace dev`
  deploys workloads into whatever context `kubectl` currently
  points at. Pointing at a prod cluster and running `devspace dev`
  will swap your prod deployment for a dev overlay. Use
  `kubectl config get-contexts`, set per-repo kubeconfigs, and
  prefer per-engineer namespaces or local kind clusters for the
  inner loop.
- **Dependency on the cluster network being stable.** A flaky VPN
  to your shared dev cluster turns the "1-second" loop into a
  "wait for sync to retry" loop. Local kind/k3d sidesteps this for
  most services; the shared cluster is for the long-tail of
  services with stateful infra dependencies.
- **6.x is a major from 5.x.** If upgrading, read the
  [migration guide](https://www.devspace.sh/docs/upgrade-guide) —
  the config schema for `images`, `dev`, and `pipelines` was
  reshaped between majors.

## Concrete example

Minimal `devspace.yaml` for a Node.js service:

```yaml
version: v2beta1
name: api

vars:
  IMAGE: ghcr.io/example/api

images:
  api:
    image: ${IMAGE}
    dockerfile: ./Dockerfile

deployments:
  api:
    helm:
      chart:
        name: ./chart       # your Helm chart, untouched
      values:
        image: ${IMAGE}

dev:
  api:
    imageSelector: ${IMAGE}
    devImage: node:20       # tools + bash on top of node base
    workingDir: /app
    sync:
      - path: ./:/app
        excludePaths:
          - node_modules
          - .git
    ports:
      - port: "3000:3000"
    logs:
      enabled: true
    command: ["npx", "nodemon", "src/index.js"]

pipelines:
  dev: |-
    create_deployments --all
    start_dev --all
  deploy: |-
    build_images --all
    create_deployments --all
```

Then:

```sh
devspace use namespace dev-alice
devspace dev                 # syncs, port-forwards :3000, tails logs
# edit src/index.js → nodemon restarts in the pod → log line on your terminal
devspace purge               # tear it back down
```
