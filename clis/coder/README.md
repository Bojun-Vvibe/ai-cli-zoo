# coder

> Snapshot date: 2026-05. Upstream: <https://github.com/coder/coder>

**A self-hosted, Terraform-driven cloud development environment
platform — workspaces are declared as Terraform modules, the
control plane provisions them on your own AWS / GCP / Azure /
Kubernetes / bare-metal hosts, and the developer connects via
SSH / VS Code Remote / a browser web-IDE through one CLI
(`coder`).**
The model: a *template* is a Terraform module that describes
what a workspace looks like (compute shape, image, persistent
disk, dotfiles, agent install); a *workspace* is one
Terraform-applied instance of that template owned by a
developer; the `coder` daemon brokers a wireguard tunnel between
the developer's laptop and the workspace VM / pod so SSH and IDE
remoting "just work" without a public IP, a corporate VPN, or a
bastion.

## Repo + version + license

- Repo: <https://github.com/coder/coder>
- Latest release: **`v2.31.11`** (2026-05-01)
- License: **AGPL-3.0** (Enterprise features under a
  commercial license; the OSS core is AGPL) —
  <https://github.com/coder/coder/blob/main/LICENSE>
- License path in repo: `LICENSE`
- Default branch: `main`
- Language: Go (server + CLI + agent), TypeScript (web UI),
  HCL (templates)

## Install

```bash
# Single-command install of the server + CLI on Linux / macOS
curl -L https://coder.com/install.sh | sh

# Start the server (Postgres-backed; use --postgres-url for prod)
coder server

# Or run dev mode against an embedded Postgres for local trial
coder server --dev

# Authenticate the CLI to a running server
coder login https://coder.example.com

# List the templates the admin published
coder templates list

# Create a workspace from a template
coder create my-dev --template=ubuntu-k8s

# Open SSH straight into it (wireguard tunnel, no bastion needed)
coder ssh my-dev

# Or open VS Code with Remote-SSH already configured
coder open my-dev --vscode

# Pull the workspace's port back to localhost
coder port-forward my-dev --tcp 8080:8080

# Suspend / resume to save cloud cost
coder stop my-dev
coder start my-dev
```

## Niche

The "**self-hosted, Terraform-shaped cloud dev environment
platform**" slot.

GitHub Codespaces, Gitpod Cloud, JetBrains Space, and Replit
Teams all solve "the developer's machine is the wrong shape for
this codebase" — the repo wants 32 GB of RAM and an A100 the
laptop does not have, the toolchain is a 40-minute install on a
fresh Mac, the codebase only builds on Linux, the secrets only
live in the corp network. Their answer is "we host it for you on
our cloud". `coder` is the answer for the orgs that *cannot* or
*will not* do that — regulated industries that need workspaces
inside their own VPC, GPU-heavy ML teams that want to amortise
their reserved capacity, sovereign clouds, on-prem-only
deployments, and teams that already have a cloud spend they want
to reuse instead of paying a per-seat SaaS markup.

The Terraform-template shape is the load-bearing decision.
Instead of a workspace being "a container the platform vendor
defined", it is "whatever Terraform can provision" — an EC2
instance, a Kubernetes pod, a GCP VM with a GPU, a bare-metal
machine via Proxmox, a `docker run` on the developer's own
laptop. That generality is why coder picks up on-prem and
GPU-cluster workflows the SaaS competitors do not.

[`devpod`](../devpod/) is the closest peer and the orthogonal
choice: devpod is *clientless* (the developer's laptop is the
control plane, the workspace lives on whatever provider the
developer points at — fly.io, AWS, a local Docker, a remote
SSH host), no team server. Pick devpod when the unit is *one
developer*; pick coder when the unit is *a team or org with
shared templates, RBAC, audit logs, and a budget owner*.

Useful for:

- **Regulated / sovereign-cloud orgs** that need workspaces
  *inside* their own VPC and audit perimeter — Codespaces /
  Gitpod Cloud are off the table for compliance reasons.
- **GPU-heavy ML teams** that already have reserved A100 / H100
  capacity and want each researcher to get a workspace on that
  pool with consistent CUDA / drivers / dataset mounts, not
  a per-seat SaaS bill on top.
- **Onboarding pain reduction** — a new hire runs `coder create
  --template=monorepo-dev` and gets the full toolchain in 10
  minutes instead of fighting `brew install` for two days.
- **Standardising the "it works on my machine" surface** — the
  template is the spec, every workspace is a Terraform-applied
  copy of it, drift is fixable with `coder update`.
- **Short-lived contributor / contractor workspaces** —
  provisioning + de-provisioning is a CLI verb, not a
  laptop-shipping process.

## Why it matters

- **Terraform-templated workspaces** — the unit of "what a dev
  environment looks like" is a Terraform module, so the full
  expressiveness of the Terraform provider ecosystem (AWS / GCP
  / Azure / Kubernetes / Docker / Proxmox / Hetzner / Vultr /
  fly.io) is in scope without bespoke platform plugins.
- **Self-hosted control plane** — one `coder server` binary +
  Postgres; runs on a single VM for a small team, scales to a
  Kubernetes-deployed HA setup with the bundled Helm chart for
  org-wide rollout. No SaaS dependency.
- **Wireguard-brokered access** — the agent inside the
  workspace dials the control plane, the developer's CLI dials
  the control plane, the control plane brokers a wireguard
  tunnel between them; no public IP on the workspace, no
  corporate-VPN dependency, the connection survives the
  workspace moving between hosts.
- **`coder ssh` / `coder open --vscode` / web IDE** — the
  developer's editor of choice (VS Code, JetBrains Gateway,
  Vim/Neovim/Helix over SSH, browser-based code-server) all
  connect to the same workspace through the same tunnel; no
  one-IDE lock-in.
- **Suspend / resume** — `coder stop` deallocates the cloud
  compute (persistent disk preserved), `coder start` brings it
  back; the cost model becomes "what fraction of the day was I
  actually working" instead of "24/7 reserved instance".
- **RBAC + audit logs + groups** — workspaces and templates
  are subject to role-based access control with full audit
  trails, suitable for the compliance posture that drove the
  team to self-host in the first place.
- **Active in 2026** — `v2.31.11` (2026-05-01) at snapshot
  time; the project ships a minor every ~3 weeks with patch
  releases mid-cycle, and the OSS repo is the same repo as the
  commercial Enterprise build (no closed fork drift).
- **Honest scope** — coder is *infrastructure*, not a
  zero-friction dev-loop tool. Standing up the control plane
  is a real systems job (Postgres, OIDC integration, template
  authoring, provider credentials), best amortised across a
  team of 10+. For one developer, [`devpod`](../devpod/) is
  the right answer; for a team of 100, coder is.
- **AGPL-3.0** — the OSS core is AGPL, which means modifying
  the server and offering it as a network service requires
  publishing the modifications. Coder Enterprise (the SSO,
  high-availability multi-region, advanced audit, and
  premium-templates surface) is sold under a commercial
  license. Self-hosting the OSS build for internal use is
  unencumbered; offering a hosted multi-tenant *coder-as-a-
  service* product to third parties triggers the AGPL clause.
- **vs Codespaces / Gitpod Cloud** — those are SaaS;
  coder is the self-hosted answer when SaaS is the wrong
  trust / cost / capacity shape.
- **vs `devpod`** — devpod is single-developer / clientless;
  coder is team / org with a control plane.
