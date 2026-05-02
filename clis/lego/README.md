# lego

> **A single-binary ACME client (Let's Encrypt, ZeroSSL, Buypass,
> Google Trust Services, any RFC 8555 CA) with first-class
> support for ~150 DNS providers** for DNS-01 wildcard issuance,
> driven entirely by CLI flags or environment variables — no
> Python, no `acme.sh` shell library, no Apache plugin. Pinned
> to **v4.27.0**
> ([LICENSE](https://github.com/go-acme/lego/blob/master/LICENSE),
> MIT).

Source: <https://github.com/go-acme/lego>

## TL;DR

`lego` is the ACME client written in Go that ships as one static
binary (~20 MB) and speaks the full RFC 8555 protocol against any
ACME-compliant CA. It handles HTTP-01, TLS-ALPN-01, and (its
killer feature) DNS-01 challenges by talking directly to your DNS
provider's API — Route 53, Cloudflare, DigitalOcean, Hetzner,
Gandi, OVH, Google Cloud DNS, Azure DNS, and roughly 145 others
— so you can issue *wildcard* certs (`*.example.com`) without
running a public-facing HTTP server. Output is plain PEM
(`certificate.crt`, `certificate.key`, `certificate.issuer.crt`,
`certificate.json`) under a deterministic directory layout, ready
to drop into nginx/HAProxy/Caddy/Traefik. Renewal is a separate
explicit subcommand (`lego renew`), designed to be wired into
cron/systemd timers/Kubernetes CronJobs rather than a daemon.

## Install

```bash
# Homebrew
brew install lego

# Go install
go install github.com/go-acme/lego/v4/cmd/lego@v4.27.0

# Static binary from GitHub Releases
curl -fsSL https://github.com/go-acme/lego/releases/download/v4.27.0/lego_v4.27.0_linux_amd64.tar.gz \
  | tar -xz -C /usr/local/bin lego

# verify
lego --version    # lego version 4.27.0
```

## One Concrete Example

```bash
# Issue a wildcard cert via Cloudflare DNS-01, save under /etc/lego.
export CLOUDFLARE_DNS_API_TOKEN='cf_api_token_with_zone_dns_edit'

sudo lego \
  --accept-tos \
  --email   ops@example.com \
  --domains 'example.com' \
  --domains '*.example.com' \
  --dns     cloudflare \
  --path    /etc/lego \
  run

# Result on disk:
# /etc/lego/certificates/example.com.crt           (fullchain PEM)
# /etc/lego/certificates/example.com.key           (private key)
# /etc/lego/certificates/example.com.issuer.crt    (CA chain only)
# /etc/lego/certificates/example.com.json          (renewal metadata)

# Renew, restart nginx only if a cert actually changed.
sudo lego \
  --accept-tos \
  --email   ops@example.com \
  --domains 'example.com' \
  --domains '*.example.com' \
  --dns     cloudflare \
  --path    /etc/lego \
  renew --days 30 --renew-hook 'systemctl reload nginx'

# Cron entry: try renewal nightly, no-op if >30 days remain.
echo '17 3 * * *  /usr/local/bin/lego --accept-tos --email ops@example.com --domains example.com --domains *.example.com --dns cloudflare --path /etc/lego renew --days 30 --renew-hook "systemctl reload nginx"' \
  | sudo tee /etc/cron.d/lego-renew
```

The discipline `lego` enforces: **issuance and renewal are CLI
invocations, not a long-lived daemon.** Cron decides when, the
hook decides what to reload, and the file layout under
`--path` is the contract with your web server. This composes
cleanly with config management (Ansible, NixOS, Chef) in a way
that daemon-style clients don't.

## Niche It Fills

**The "I need a real cert from a real CA, scriptable, with
wildcard support, without inviting Python or a long-lived agent
into my server" client.** Most useful when the renewal trigger
should be cron + a reload hook — nginx, HAProxy, Postfix, MQTT
brokers, internal API gateways behind a corporate split-horizon
DNS where HTTP-01 isn't reachable from the public internet but
the DNS zone is API-controllable.

## Vs Already Cataloged

- **Vs [`step`](../step/) / [`step-cli`](../step-cli/):** smallstep
  is a CA + ACME-client + SSH-CA + JWT toolkit; if you're running
  your *own* internal CA, `step-ca` issues and `step ca certificate`
  fetches. `lego` is purely a *client* against a public ACME CA.
  Use `step` for internal mTLS PKI; use `lego` to get a
  publicly-trusted cert from Let's Encrypt for the edge.
- **Vs [`caddy`](../caddy/):** Caddy includes ACME natively and
  renews automatically — if Caddy *is* your web server, you don't
  need `lego`. `lego` is for the case where the TLS terminator is
  nginx, HAProxy, Envoy, or some appliance that takes PEM files
  on disk.
- **Vs [`cloudflared`](../cloudflared/):** Cloudflare Tunnel
  bypasses the cert problem entirely (Cloudflare terminates TLS
  at the edge, the tunnel carries plaintext to your origin). If
  you're already on Cloudflare and willing to terminate there,
  you don't need a cert on the box. `lego` is for "I want to
  terminate TLS on my own server with my own cert."
- **Vs [`mkcert`](../mkcert/):** `mkcert` issues *locally trusted*
  development certs from a private root added to your system
  trust store. `lego` issues *publicly trusted* production certs
  from a real CA. Different jobs entirely; mkcert for laptop
  dev, lego for the edge.
- **Vs `certbot`:** the canonical comparison. `certbot` is
  Python, ships with web-server plugins (nginx, apache) that
  rewrite your config in-place, and tracks an account/cert
  database under `/etc/letsencrypt/`. `lego` is one Go binary,
  has *no* web-server plugins (it just writes PEM files), and is
  stateless beyond `--path`. Pick `certbot` for Ubuntu boxes
  with one nginx vhost where its auto-config is convenient; pick
  `lego` for everything else (containers, NixOS, immutable
  infra, exotic web servers, cross-CA setups, anywhere a Python
  runtime is unwanted).
- **Vs `acme.sh`:** pure bash, also supports DNS-01 with many
  providers. `acme.sh` shines on minimal containers (no Go
  binary needed, just curl/openssl) and has even broader DNS
  provider coverage. `lego` wins on type safety, error messages,
  static binary distribution, and library reusability (Traefik
  embeds `lego` directly for its built-in ACME).

## Caveats

- DNS-01 requires API credentials with permission to write TXT
  records on the zone — those credentials end up in env vars or
  config files on the host. Scope the API token to the *single*
  zone (Cloudflare's "Zone:DNS:Edit" scoped to one zone, AWS
  IAM policy with `Resource` pinned to one hosted-zone ARN);
  do not use account-wide tokens.
- DNS propagation timing matters: lego polls authoritative NS
  for the TXT record before completing the challenge, but if
  your zone is behind a slow secondary or a CDN-wrapped DNS,
  bump `--dns.resolvers` to the authoritative servers directly
  and consider `--dns.disable-cp` (disable challenge propagation
  preflight) if you have ironclad certainty about timing.
- `--path` is the source of truth for renewal — losing it means
  losing the account key and having to re-register. Back it up
  (it's small: tens of KB), or store it on persistent volumes
  for containerized renewers.
- Let's Encrypt rate limits are per-account and per-domain
  (50 certs per registered domain per week, 5 duplicate-cert
  per week, 300 pending auths). When iterating on lego config,
  point `--server` at the staging endpoint
  (`https://acme-staging-v02.api.letsencrypt.org/directory`)
  until the flow is correct, then flip to prod.
- `lego` does not reload your web server — that's the
  `--renew-hook` callback's job. A common bug is forgetting the
  hook and seeing renewals succeed but nginx serving the old
  cert until the next config reload.
- The library and CLI are versioned together; pinning the
  binary to a specific tag (as above) is wise because plugin
  configuration env-var names occasionally change between
  major versions.
