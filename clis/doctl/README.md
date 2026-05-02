# doctl

> **The official DigitalOcean command-line client — one Go
> binary that wraps the entire DigitalOcean API: Droplets,
> Kubernetes (DOKS), App Platform, Spaces (S3-compatible),
> Volumes, Databases (managed Postgres / MySQL / Redis /
> Kafka / MongoDB), Load Balancers, Floating / Reserved IPs,
> Firewalls, VPCs, Container Registry, Functions (serverless),
> Monitoring, and the GenAI Platform — every API surface is a
> verb-noun subcommand (`doctl compute droplet create`,
> `doctl kubernetes cluster kubeconfig save`, `doctl apps
> create --spec app.yaml`, `doctl databases create`), all
> output is `--format` selectable (table / JSON / no-header /
> per-column) so it pipes into `jq` / `awk` like any other Unix
> tool.** Pinned to **v1.155.0** (released 2026-04-16),
> [LICENSE.txt](https://github.com/digitalocean/doctl/blob/main/LICENSE.txt),
> Apache-2.0.

Source: <https://github.com/digitalocean/doctl>

## TL;DR

`doctl` is to DigitalOcean what `aws` is to AWS, `gcloud` is
to GCP, and `flyctl` is to Fly.io — the **single canonical CLI
the vendor ships, owns, and gates feature parity against the
web console on**. Authentication is one `doctl auth init`
(paste a Personal Access Token from
`https://cloud.digitalocean.com/account/api/tokens`) which
writes a context to `~/Library/Application Support/doctl/config.yaml`
(macOS) or `~/.config/doctl/config.yaml` (Linux); multi-account
work is `doctl auth init --context personal` /
`doctl auth switch --context work`. The verb-noun grammar
(`doctl <resource-group> <resource> <action>`) covers every
public API endpoint without resorting to raw `curl`, and
`--format` + `--no-header` make every list command shell-pipe
shaped (`doctl compute droplet list --format ID,Name,PublicIPv4
--no-header | awk '$3==""'` finds Droplets without a public
IP). For GitOps-shaped workflows, App Platform takes a
declarative `app.yaml` spec (`doctl apps create --spec
app.yaml` / `doctl apps update <id> --spec app.yaml`) so the
deployment unit is a committed file not a click-trail.

## Install

```bash
# Homebrew (macOS / Linux)
brew install doctl

# Snap (Ubuntu)
sudo snap install doctl

# Static binary from GitHub Releases (linux / macOS / windows, amd64 / arm64)
curl -fsSL -o /tmp/doctl.tar.gz \
  https://github.com/digitalocean/doctl/releases/download/v1.155.0/doctl-1.155.0-darwin-arm64.tar.gz
tar -xzf /tmp/doctl.tar.gz -C /usr/local/bin doctl

# Docker (great for CI — no toolchain pin)
docker run --rm -it -e DIGITALOCEAN_ACCESS_TOKEN=$DO_TOKEN \
  digitalocean/doctl:1.155.0 compute droplet list

# Verify
doctl version    # doctl version 1.155.0-release

# First-time auth
doctl auth init                              # paste PAT
doctl account get                            # confirms identity
```

## One Concrete Example

```bash
# Spin up a 2 vCPU / 4 GB Droplet in NYC3 from a snapshot,
# attach a Reserved IP, and dump the kubeconfig of an existing
# DOKS cluster — all scripted, no console.

# 1. Find your SSH key fingerprint already uploaded to DO
doctl compute ssh-key list --format ID,Name,FingerPrint --no-header

# 2. Create the Droplet (returns JSON with the new ID)
DROPLET_ID=$(doctl compute droplet create web-prod-01 \
  --region nyc3 --size s-2vcpu-4gb \
  --image ubuntu-24-04-x64 \
  --ssh-keys 12:34:ab:cd:... \
  --tag-name web,prod \
  --enable-monitoring --enable-ipv6 \
  --wait \
  --format ID --no-header)

# 3. Reserve a public IP and assign it
doctl compute reserved-ip create --region nyc3
doctl compute reserved-ip-action assign 203.0.113.42 $DROPLET_ID

# 4. Pull the DOKS kubeconfig into your local ~/.kube/config
doctl kubernetes cluster kubeconfig save prod-cluster

# 5. Tear it all down at end of demo
doctl compute droplet delete $DROPLET_ID --force
doctl compute reserved-ip delete 203.0.113.42 --force
```

For App Platform (declarative deploy):

```yaml
# app.yaml
name: my-api
region: nyc
services:
  - name: api
    git:
      repo_clone_url: https://github.com/me/my-api.git
      branch: main
    build_command: go build -o bin/api ./cmd/api
    run_command: ./bin/api
    instance_size_slug: basic-xxs
    instance_count: 2
    http_port: 8080
    health_check:
      http_path: /healthz
```

```bash
doctl apps create --spec app.yaml          # initial deploy
doctl apps list --format ID,Spec.Name,DefaultIngress
doctl apps update <app-id> --spec app.yaml # subsequent rollout
doctl apps logs <app-id> --type=run --follow
```

## License

[Apache-2.0](https://github.com/digitalocean/doctl/blob/main/LICENSE.txt),
SPDX `Apache-2.0`.

## Niche / positioning

The **first-party CLI for DigitalOcean** — orthogonal to every
other cloud CLI in this catalog because it talks to a different
provider's API. Pick `doctl` over hand-rolled `curl` against
`api.digitalocean.com/v2/...` for typed flags, paginated `list`
helpers, `--wait` for async actions (Droplet create / snapshot /
resize block until status flips), and `--format` for shell-pipe
output. Pick over [`flyctl`](../flyctl/) / [`heroku`](../heroku/) /
[`railway`](../railway/) / [`render-cli`](../render-cli/) only
when the deployment target is *DigitalOcean specifically* —
those are different PaaS / IaaS providers and the comparison is
"which cloud", not "which CLI". For Kubernetes management,
`doctl kubernetes` covers DOKS-specific operations (cluster
create, node pool resize, kubeconfig wiring); once the
kubeconfig is local, [`kubectl`](../kubectl/) / [`k9s`](../k9s/)
take over for in-cluster work. For S3-compatible Spaces, you can
either use `doctl compute cdn` for CDN-layer ops or point any
S3 client (`aws s3 --endpoint-url`, [`s3cmd`](../s3cmd/),
[`rclone`](../rclone/)) at the Spaces endpoint. Skip when the
workload is on AWS / GCP / Azure / OCI / Linode / Vultr — the
right CLI is the vendor's own.
