# kamal

> **Single-binary deploy tool that ships a Dockerized app onto any
> set of SSH-reachable Linux hosts with zero-downtime rollouts and a
> built-in HTTPS proxy.**
> A Ruby gem (`kamal`) that takes a `config/deploy.yml`, builds your
> image (locally or on a remote builder), pushes it to a registry,
> SSH-streams the new container onto each target host, and uses its
> own `kamal-proxy` to drain old containers and route traffic to new
> ones — no Kubernetes, no managed PaaS, just `kamal deploy`. Pinned
> to **v2.11.0** (released 2026-03-18, SPDX: `MIT`,
> [MIT-LICENSE](https://github.com/basecamp/kamal/blob/main/MIT-LICENSE)).

Source: <https://github.com/basecamp/kamal>

## TL;DR

`kamal deploy` takes the app you can already run with `docker run`,
pushes it to N bare VMs (Hetzner, EC2, DigitalOcean droplets, an
on-prem rack — anything with SSH and Docker), and orchestrates a
rolling cutover with health checks and automatic rollback. The
control plane is your laptop or CI runner; the data plane is plain
Docker on plain Linux. There is nothing else to install on the
servers besides Docker — `kamal` bootstraps `kamal-proxy` itself
and writes systemd units as needed.

## Install

```sh
# Recommended (per-app, version-pinned in Gemfile)
bundle add kamal --version "~> 2.11"
bundle exec kamal --version    # 2.11.0

# Or system-wide
gem install kamal -v 2.11.0
kamal --version                # 2.11.0

# One-time scaffold for an existing app
kamal init                     # writes config/deploy.yml + .kamal/
```

## License

MIT — unrestricted. Safe to embed in CI, ship as part of a
self-hosted PaaS template, or vendor into a deploy-tooling
monorepo. The companion `kamal-proxy` (Go) is also MIT under
[basecamp/kamal-proxy](https://github.com/basecamp/kamal-proxy).

## Primary use case

You have a small-to-mid Rails / Django / Node / Go app that wants
to live on three to thirty Linux VMs you already pay for. You do
not want to run a Kubernetes cluster, you do not want a managed
container PaaS bill, and you do not want to write 800 lines of
Ansible. `kamal deploy` gives you: build → push → rolling restart
across the fleet → automatic Let's Encrypt certs via
`kamal-proxy` → `kamal logs` / `kamal app exec` for ops. A new
host joins the fleet by adding one line to `deploy.yml` and
running `kamal setup`.

## What it competes with

- **Kubernetes (`kubectl`, `helm`,
  [`argocd`](../argocd/), [`kustomize`](../kustomize/))** —
  industry-standard at scale, dramatically over-built for a
  3-VM Rails app. Pick k8s when the workload count and team size
  justify the operator burden; pick `kamal` when they don't.
- **Heroku / Fly / Render / Railway (cf. [`flyctl`](../flyctl/))**
  — managed PaaSes that hide the VMs entirely. Pick a PaaS for
  zero-ops; pick `kamal` when you specifically want to *own*
  the hosts (cost, data-residency, regulated workloads,
  GPU-on-bare-metal) without giving up the deploy ergonomics.
- **[`docker compose`](https://docs.docker.com/compose/) +
  bespoke SSH scripts** — what most "we just rsync it" setups
  evolve into. `kamal` is roughly that, but with rolling
  deploys, registry-based image distribution, secrets handling,
  and a proxy already wired up.
- **Capistrano / Mina / Ansible playbooks** — pre-container
  deploy stacks. `kamal` replaces them for any app you can
  package as an OCI image.
- **[`dagger`](../dagger/) / [`earthly`](../earthly/) /
  [`skaffold`](../skaffold/)** — pipeline tooling, not deploy
  targets. They build; `kamal` deploys what they build.

## AI-native angle

A coding agent that owns "feature → green CI → production" needs
a deploy target whose surface is small enough to reason about
end-to-end:

- **One config file is the whole deploy graph.** `config/deploy.yml`
  declares servers, image, env, healthcheck, proxy hostnames,
  rollout policy. An agent can read the file, propose a diff,
  and explain the blast radius of each line. Compare with k8s,
  where the deploy graph is spread across Helm charts, kustomize
  overlays, ArgoCD apps, and CRDs.
- **Deploys are reversible by design.** `kamal rollback` flips
  back to the previous image tag in seconds. An agent that ships
  a bad change can roll itself back without paging a human, and
  the recovery action is one command.
- **Idempotent verbs.** `kamal deploy`, `kamal redeploy`,
  `kamal app boot`, `kamal app stop`, `kamal proxy reboot` are
  all safe to re-run. Agents tolerate retries cleanly.
- **`kamal app exec` is a bounded shell.** An agent triaging a
  prod incident can run `kamal app exec --reuse "rails runner
  …"` to inspect state without SSH-ing onto the box. The blast
  radius is exactly "one container exec," easy to log and
  review.
- **Secrets stay in the broker.** `kamal` reads secrets from
  1Password / Bitwarden / `op` / env at deploy time and pushes
  them to the host as Docker env. They never have to land in a
  prompt or the agent's chat history.

## Caveats

- **Stateful services are out of scope.** Postgres / Redis /
  S3-equivalent / Kafka are *not* what `kamal` deploys. It
  assumes you point your app at managed (or separately-deployed)
  data services. Treat the VM fleet as cattle for stateless apps
  and a sidecar accessory for everything else.
- **Single registry, single image per app.** The model is one
  app, many hosts. Multi-app monorepos work but you run `kamal`
  once per app; there is no top-level "deploy the whole org."
- **`kamal-proxy` is opinionated.** It does HTTP(S), websockets,
  Let's Encrypt, sticky sessions; it is not a Layer-7 mesh, not
  an ingress controller, not Envoy. If you need request-shaping
  beyond "route by host," you front `kamal-proxy` with a real
  load balancer or replace it.
- **SSH is load-bearing.** Deploys go over SSH from the runner
  to each host. If your CI cannot reach production via SSH, you
  need a deploy bastion or a self-hosted runner inside the VPC.
- **Ruby runtime on the operator side.** The CLI itself is Ruby;
  the servers do not need Ruby, only Docker. CI images that
  don't already have Ruby need to install it (or use the
  official `ghcr.io/basecamp/kamal` runner image).

## Concrete example

`config/deploy.yml` for a small Rails app on three Hetzner VMs:

```yaml
service: acme-web
image: acme/web

servers:
  web:
    hosts:
      - 5.75.10.11
      - 5.75.10.12
      - 5.75.10.13
    options:
      restart: unless-stopped

proxy:
  ssl: true
  host: app.acme.example
  healthcheck:
    path: /up
    interval: 3
    timeout: 10

registry:
  server: ghcr.io
  username: acme-deploy
  password:
    - KAMAL_REGISTRY_PASSWORD

env:
  clear:
    RAILS_ENV: production
    RAILS_LOG_TO_STDOUT: "1"
  secret:
    - RAILS_MASTER_KEY
    - DATABASE_URL
    - REDIS_URL

asset_path: /rails/public/assets

builder:
  arch: amd64
  cache:
    type: registry
```

```sh
kamal setup        # one-time: bootstrap proxy, login, first deploy
kamal deploy       # subsequent rollouts (zero-downtime)
kamal rollback     # revert to previous image tag
kamal app logs -f  # tail logs across the fleet
kamal app exec --interactive --reuse "bin/rails console"
```

The whole loop — code change to live on three boxes — is one
command, and the rollback is also one command.
