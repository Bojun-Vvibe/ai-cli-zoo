# flyctl

- **Repo:** https://github.com/superfly/flyctl
- **Version:** v0.4.45
- **License:** [LICENSE](https://github.com/superfly/flyctl/blob/master/LICENSE) (Apache-2.0)
- **Category:** PaaS / Deployment CLI

## What it is

`flyctl` (often invoked as `fly`) is the official command-line client for
[Fly.io](https://fly.io), a public cloud that runs your containers as
Firecracker microVMs in 30+ regions. The CLI is the *primary* control
surface for Fly: it builds your image (locally or via a remote builder),
ships it to Fly's registry, declares the desired Machines / volumes /
secrets, runs migrations and SSH sessions into running VMs, and wires up
private networking, Anycast IPs, and TLS certificates.

## Why it's interesting

- **Single-command deploy from a Dockerfile or buildpack.** `fly launch`
  inspects your repo, generates a `fly.toml`, provisions an app, and
  `fly deploy` ships the next revision — no separate IaC tool needed for
  the basics.
- **Machines API as a primitive.** `fly machine run` boots a microVM in
  ~1s per region for cron-like jobs, ephemeral workers, or
  pull-request preview environments without a long-running app object.
- **Built-in Postgres, Redis (Upstash), and Tigris S3-compatible storage**
  attach via `fly postgres create` / `fly redis create` / `fly storage
  create` and are wired into your app as secrets automatically.
- **Multi-region anycast for free.** `fly scale count 2 --region iad,fra`
  puts replicas behind a single Anycast IPv4/IPv6 with regional routing —
  the closest Fly edge terminates TLS and forwards.
- **`fly ssh console` and `fly proxy`** give you direct shell into a
  Machine and a local TCP forwarder to private 6PN addresses, which is the
  feature that turns Fly into a viable replacement for "I just need a tiny
  VPS with a real network."

## Install

```bash
# macOS / Linux installer
curl -L https://fly.io/install.sh | sh

# Homebrew
brew install flyctl

# Windows
# iwr https://fly.io/install.ps1 -useb | iex

fly version    # fly v0.4.45 ...
fly auth login
```

## Examples

```bash
# bootstrap a new app from the current repo
fly launch --no-deploy

# deploy current directory using the Dockerfile / buildpack
fly deploy --remote-only

# scale to two machines across two regions
fly scale count 2 --region iad,fra

# attach a managed Postgres
fly postgres create --name myapp-db --region iad
fly postgres attach myapp-db

# tail logs across all regions
fly logs

# shell into a running Machine
fly ssh console

# run a one-shot migration job in a fresh Machine that exits when done
fly machine run . --rm --command "bin/rails db:migrate"
```

## Use when

- You want a "git push to deploy"-feeling PaaS that still gives you real
  containers, real volumes, real private networking, and real multi-region
  routing — without writing Terraform or running your own Kubernetes.
- You need globally-distributed low-latency replicas of a small service
  (edge auth, image transformation, websocket fan-out) and the
  per-Machine billing model fits better than reserving cloud VMs in N
  regions.
- You want preview environments per pull-request: `fly machine run` is
  cheap and fast enough that a CI job can spin up and tear down a real
  microVM per PR.

## Use something else when

- You need a fully-managed Kubernetes API surface — Fly's Machines are
  *not* Kubernetes; reach for GKE / EKS / AKS or `kind`/`k3d` locally.
- Your workload is steady-state and large (single tenant >32 GB RAM
  always-on): a reserved cloud VM or a bare-metal provider is cheaper.
- You want a true serverless function model with sub-second cold-start
  scale-to-zero for HTTP — Cloudflare Workers (see
  [`wrangler`](../wrangler/)) or AWS Lambda fit better.

## Alternatives

- [`wrangler`](../wrangler/) — Cloudflare Workers / Pages / R2 CLI for
  edge-runtime serverless.
- `railway` (Railway CLI), `render`, `flightcontrol` — competing
  developer-PaaS clients.
- `kubectl` + a managed Kubernetes — if you have already standardised on
  Kubernetes manifests, Fly's bespoke `fly.toml` may not be worth it.
