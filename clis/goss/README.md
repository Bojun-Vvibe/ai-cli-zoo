# goss

> **Server validation as YAML — assert what a host should look
> like, then run the assertions in milliseconds** — a single Go
> binary that turns "is sshd listening on 22, is the postgres
> user UID 999, is `/etc/nginx/nginx.conf` mode 0644 and owned by
> root, does `curl localhost/health` return 200" into a
> declarative test file you commit next to your IaC. Pinned to
> **v0.4.9** (released 2024-09-13,
> [LICENSE](https://github.com/goss-org/goss/blob/master/LICENSE),
> Apache-2.0).

Source: <https://github.com/goss-org/goss>

## TL;DR

`goss` is the answer when you have provisioned a host (Ansible /
cloud-init / a `Dockerfile` / a Packer build) and you need a
machine-checkable answer to "is this server actually in the state
I claimed". One YAML file enumerates the expected `package`,
`service`, `port`, `user`, `group`, `file`, `command`, `dns`,
`http`, `process`, `mount`, `interface`, `kernel-param`,
`addr`, `matching` resources. `goss validate` checks every
assertion against the live system in parallel and exits non-zero
on the first failure with a structured report (TAP / JUnit /
JSON / nagios / rspecish / silent). The companion `dgoss` wraps
the same flow around `docker run` for image testing in CI, and
`goss serve` exposes the validation result as an HTTP
healthcheck endpoint your load balancer can poll.

## Install

```bash
# one-liner installer (Linux / macOS / *BSD)
curl -fsSL https://goss.rocks/install | sh

# Homebrew (macOS / Linux)
brew install goss

# Linux package managers
# Arch (AUR): yay -S goss-bin
# Nix: nix-env -iA nixpkgs.goss

# Static binary (any OS)
# https://github.com/goss-org/goss/releases

# verify
goss --version    # goss version v0.4.9
```

## Examples

```bash
# bootstrap a goss.yaml from the live system (auto-discover what's there)
goss autoadd nginx          # adds package + service + process + listening ports for nginx
goss add port tcp:80        # add an explicit assertion
goss add http http://localhost/health
cat goss.yaml               # commit this next to your Ansible / Packer / Dockerfile

# run the assertions on this host
goss validate
goss validate --format=tap        # TAP for CI consumers
goss validate --format=junit > junit.xml
goss validate --retry-timeout=30s --sleep=1s   # wait for slow boot

# wrap a docker image build in the same loop
dgoss run -d -p 8080:80 nginx:1.27        # builds container, waits, runs goss inside

# turn the validation into a long-lived HTTP healthcheck
goss serve --format=json --listen-addr=:8080 &
curl -s localhost:8080/healthz | jq .

# render the asserted state of a host as a goss.yaml without writing anything
goss render
```

## Use when

- You ship **golden images** (Packer / EC2 AMIs / GCP images /
  bootable container images) and need a CI gate that fails the
  build if the resulting image is missing a daemon, has the wrong
  file permissions, or exposes an unexpected port — `dgoss` /
  `kgoss` / plain `goss validate` slot directly into the pipeline.
- You want a **server-side healthcheck that knows about more than
  one HTTP endpoint** — a real "is this host actually serving
  traffic correctly" check covers process, port, file, DNS, and
  HTTP simultaneously and `goss serve` exposes the rolled-up
  result as one URL the LB can poll.
- You run **Ansible / Salt / Chef / Puppet** and want the
  *outside-in* test that catches what the converge step missed
  (a service that started but is failing, a config file that was
  templated but is `0600` instead of `0644`, a port the firewall
  is blocking) — goss runs in seconds, declarative, idempotent.
- You promote container images through environments and want a
  smoke-test layer that runs *inside* the image at build time
  rather than against a deployed instance — `dgoss` is that layer.
- Pair with [`hadolint`](../hadolint/) (lint the Dockerfile),
  [`dockle`](../dockle/) (audit the resulting image's
  configuration), [`trivy`](../trivy/) / [`grype`](../grype/)
  (scan for CVEs), goss (assert runtime behaviour) — four
  orthogonal gates for one image build.

Skip `goss` when a single `curl /health` covers the question, when
you actually need full integration tests (use Bats / pytest / Go
test against a real client), or when the host is so dynamic
(autoscaling, ephemeral) that the assertions you would write are
mostly "this thing exists in some form" — at that point a
metrics-based readiness probe wins.
