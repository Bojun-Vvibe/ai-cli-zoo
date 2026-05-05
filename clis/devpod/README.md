# devpod

> **Client-only, open-source dev-environment manager that takes a
> standard `devcontainer.json` and spins it up against any backend
> — local Docker / Podman, an SSH host, a Kubernetes cluster, AWS
> / GCP / DigitalOcean / a self-hosted VM — with the same `devpod
> up <repo>` command.** One Go binary; no central control plane,
> no SaaS account, no vendor lock-in. Pinned to **v0.6.15** (commit
> `33d20ff8806a3fee86d8f56ed50db6108b945fc2`),
> MPL-2.0
> ([LICENSE](https://github.com/loft-sh/devpod/blob/main/LICENSE)).

- **Repo:** https://github.com/loft-sh/devpod
- **Latest version:** v0.6.15
- **License:** MPL-2.0 (`LICENSE` at repo root, SPDX `MPL-2.0`)
- **Category:** `dev-environments` / `containers` /
  `devcontainer-spec`
- **Language:** Go

## What it does

`devpod` is a CLI implementation of the open
[devcontainer spec](https://containers.dev) — the same JSON schema
your `.devcontainer/devcontainer.json` already uses — that
decouples the *spec* from the *runtime*. Point `devpod up
github.com/owner/repo` at a repo containing a `devcontainer.json`
and it: clones the repo, resolves any base image / `features` /
`customizations`, builds (locally or remotely) a workspace
container, mounts the source tree, runs the `postCreateCommand` /
`postStartCommand` / `postAttachCommand` hooks, and connects your
chosen IDE (VS Code remote, JetBrains Gateway, browser-based
openvscode-server, or just `devpod ssh <name>` for a bare shell).
The "where" is a *provider* — a self-contained binary plug-in that
implements `init`, `create`, `start`, `stop`, `delete`, `command`
verbs against a target. First-party providers cover `docker`,
`kubernetes`, `ssh`, `aws`, `gcp`, `azure`, `digitalocean`, and a
generic `terraform` adapter; third-party providers exist for
Hetzner, OVH, Civo, Scaleway, and bare-metal hypervisors. A
provider switch is `devpod provider use kubernetes` — the same
workspace definition lifts off the laptop's Docker daemon and
materialises as a Pod in a remote cluster with the same code, the
same tools, the same hooks.

The control surface is intentionally local: `devpod` keeps state
in `~/.devpod/` (workspace metadata, provider configs, SSH keys),
talks directly to the backend's API (Docker socket, kubeconfig,
cloud SDK), and never phones home. Workspaces are
list-able / `up` / `stop` / `delete`-able from the CLI, browseable
from the optional desktop app, and SSH-aliased so `ssh <workspace>`
from any other tool (`scp`, `rsync`, `mosh`, your editor's "Remote
SSH" mode) works without remembering the underlying connection
string. Pre-built workspaces are shareable: `devpod machine create
… --provider aws` provisions a long-lived EC2 host that subsequent
`devpod up` invocations reuse instead of spinning up a fresh VM
per workspace.

## When to pick it / when not to

Pick `devpod` when your team already authors `devcontainer.json`
files (or wants to) and the open-source / self-hosted / vendor-
neutral execution path matters: a single spec runs on a developer's
local Docker, a shared on-prem Kubernetes cluster, a per-engineer
EC2 instance, or a CI runner with no per-environment YAML
duplication. Pick it when "spin up a fresh, reproducible
environment for a contributor / interview / bug repro / OSS
maintainer review" is a recurring workflow — `devpod up
github.com/owner/repo --ide vscode` is the one-liner. Pick the
Kubernetes provider for "team of N developers who each get a Pod
in the dev cluster, scaled to zero overnight" when paying for a
hosted environment-as-a-service is undesirable. Pick the SSH
provider for "I have a beefy home / lab machine; my laptop is the
client" without any container-runtime install on the laptop.

Skip it when your editor's built-in dev-container support is
enough: VS Code's "Reopen in Container" command works fine for the
*solo developer, local Docker only* case without a separate CLI.
Skip it when your backend is a fully-managed
environment-as-a-service that already wraps the spec for you
(Coder, Gitpod, the major code-hosting forges' first-party
remote-environment products) — `devpod` competes with the *runtime*
layer of those products, not the curated UX. Skip it when the
workload does not fit a container or a single VM (heterogeneous
multi-node setups, GPU clusters with sharded models, tight
hypervisor coupling) — pair with [`tilt`](../tilt/) /
[`skaffold`](../skaffold/) / [`kustomize`](../kustomize/) for the
multi-service Kubernetes-native shape instead. Skip it when
"isolation per task" really means "ephemeral CI runner" — that is
the natural shape of GitHub Actions / GitLab CI / Drone / Buildkite
runners, not a developer-facing workspace tool.

Vs already cataloged: orthogonal to [`devbox`](../devbox/) /
[`flox`](../flox/) / [`mise`](../mise/) / [`pkgx`](../pkgx/) /
[`pixi`](../pixi/) / [`nixpacks`](../nixpacks/) — those
declaratively pin a *toolchain* (Node, Python, Go, Rust + libs) on
the host without containers; `devpod` materialises a *whole
container or VM* per workspace. Compose: a `devcontainer.json`
that runs `devbox shell` as its `postCreateCommand` gets you
`devpod`-managed lifecycle + `devbox`-managed toolchain. Distinct
from [`distrobox`](../distrobox/) (rootless container *as a
second-distro shell* on the same host — overlapping but
distrobox is host-mounted-everything by default, devpod is
isolated-by-default with explicit mount syntax). Distinct from
[`colima`](../colima/) / [`lima`](../lima/) / [`orbstack`](../) /
[`podman`](../podman/) — those provide the *container engine*
underneath; `devpod`'s docker provider sits on top of them.
Distinct from [`daytona`](../daytona/) — daytona is the closest
peer (also a self-hosted dev-environment manager with a
devcontainer spec frontend) and ships a server component for
team-shared workspaces; pick `devpod` for the
client-only / no-server / per-user-CLI shape, pick `daytona` for
the central-team-server shape.

## Example invocations

```bash
# Configure providers — one-time setup
devpod provider add docker      # local Docker / Podman
devpod provider add kubernetes  # uses your current kubeconfig
devpod provider add ssh         # remote host via SSH

# Choose the active provider (per-workspace override available)
devpod provider use docker

# Spin up a workspace from a repo with a devcontainer.json
devpod up https://github.com/owner/example-repo

# … with a specific IDE
devpod up https://github.com/owner/example-repo --ide vscode
devpod up https://github.com/owner/example-repo --ide jetbrains
devpod up https://github.com/owner/example-repo --ide openvscode  # browser

# Switch the same workspace to a remote backend without redefining it
devpod provider use kubernetes
devpod up <workspace-name>      # rebuilds in the cluster

# SSH into the workspace from any other tool
ssh <workspace-name>            # devpod registers an SSH alias
rsync -av <workspace-name>:/workspace/dist/ ./local-dist/

# List, stop, delete
devpod list
devpod stop  <workspace-name>
devpod delete <workspace-name>

# Reuse a long-lived backing VM across many workspaces
devpod machine create dev-vm --provider aws \
  --option DiskImage=ami-... --option MachineType=m6i.large
devpod up <repo> --machine dev-vm

# Verify
devpod version    # devpod v0.6.15
```

## Caveats

- **Provider quality varies.** First-party `docker` /
  `kubernetes` / `ssh` are the well-trodden paths; cloud
  providers work but expect provider-specific quirks (IAM scopes,
  region defaults, cleanup edge cases) — read the provider's
  README before standing up team-shared infrastructure on it.
- **Pre-1.0.** Workspace-state schema and provider contracts have
  changed historically; pin the version per-machine and treat
  upgrades as flag days (`devpod stop` before, `devpod up` after).
- **`devcontainer.json` parity is high but not 100%.** Some less-
  common features / lifecycle hooks supported by the reference
  CLI are still in flight here; if a workspace works in the
  reference implementation but not in `devpod`, file a parity
  issue rather than rewriting the spec.
- **Local Docker provider is a thin wrapper.** Networking,
  port-forwarding, and bind-mount semantics are whatever the
  underlying Docker / Podman engine does — `devpod` does not
  paper over engine-level differences (e.g. macOS volume-mount
  performance under VirtioFS vs gRPC-FUSE is unchanged).
- **Browser-IDE mode runs an `openvscode-server` instance** —
  authentication is a query-string token by default; do not
  expose the workspace's IDE port to the public internet without
  a reverse proxy and proper auth in front of it.
