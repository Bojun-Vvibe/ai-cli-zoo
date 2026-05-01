# traefik

- **Repo:** https://github.com/traefik/traefik
- **Version:** v3.3.4 (current stable on the v3.x line)
- **License:** MIT — see [`LICENSE.md`](https://github.com/traefik/traefik/blob/master/LICENSE.md)
- **Language:** Go (single static binary; embedded YAEgo middleware engine; netlink/eBPF socket plumbing on Linux; ACME client built in)
- **Install:** `brew install traefik` · `docker run traefik:v3.3` · `apt install traefik` (via OpenSUSE OBS) · or grab the static binary from the GitHub releases page (`traefik_v3.3.4_linux_amd64.tar.gz`); the same binary serves as edge router, dashboard, and ACME client — no separate sidecar required

## Overview

`traefik` is a cloud-native HTTP / TCP / UDP reverse proxy
and load balancer that **discovers its own configuration**
from the runtime it is embedded in, instead of asking an
operator to maintain a parallel routing file. Point it at
a Docker socket, a Kubernetes API server (Ingress, IngressRoute
CRD, Gateway API), a Consul / Nomad / ECS / Zookeeper / etcd /
Redis store, or a directory of YAML / TOML files, and it
materialises the router → middleware → service graph by
reading labels / annotations / KV keys directly. New container
comes up with `traefik.http.routers.api.rule=Host('api.example.com')`
labels — within a few hundred ms it is reachable on the
public listener with TLS terminated, ACME-issued, and rate-
limited per the same labels. Containers go away — routes
disappear without an operator touching a config file.
The middleware library covers the usual edge concerns
(stripPrefix / addPrefix, basicAuth, forwardAuth, rate-
limit, circuitBreaker, retry, headers, IPWhiteList,
inFlightReq, compress, errors, redirectScheme,
redirectRegex, contentType, plugin) as composable units
that attach to routers, not to services, so the same
backend can be reached through different middleware
chains for staff vs public traffic. ACME / Let's Encrypt
is built in: `--certificatesresolvers.le.acme.email=…`
plus a TLS challenge or DNS-01 provider gives you
auto-renewing certs with no cron job. The dashboard
(opt-in, `--api.dashboard=true`) exposes the live router
graph and middleware chain as a browsable UI for
debugging "why is this request not routing?" without
turning on access-log spelunking.

## Niche

**The reverse-proxy you point at Docker / Kubernetes /
Consul and walk away — every service publishes its own
routing rules as labels / annotations and traefik picks
them up without an edit-and-reload step, with built-in
ACME, a composable middleware library, HTTP/3 support,
and a built-in observability surface.** The role is "the
edge router for an environment where containers /
services come and go faster than an operator can edit
nginx.conf, and you want TLS, retries, rate-limit,
basicAuth, and dashboards out of the box without
bolting on cert-manager + ingress-nginx + metrics-server
+ a Lua plugin pile." Competing universe: nginx /
HAProxy / Caddy / Envoy. See comparisons below.

## When to use

- You run a Docker / Compose / Swarm host and want
  per-container routing as labels: a service's
  `compose.yaml` declares `traefik.http.routers.foo.rule=Host('foo.dev')`
  and traefik routes it within a second of `docker compose up`.
- You run Kubernetes and want an ingress controller
  with a richer CRD model than `Ingress`: traefik's
  `IngressRoute` / `IngressRouteTCP` / `IngressRouteUDP`
  CRDs expose middleware and TLS options inline,
  plus full Gateway API v1 support (`HTTPRoute`,
  `TLSRoute`, `TCPRoute`, `GRPCRoute`).
- You want **ACME / Let's Encrypt baked in** without
  running cert-manager: `--certificatesresolvers.le.acme.dnschallenge.provider=cloudflare`
  plus a `CF_DNS_API_TOKEN` env var gets you wildcard
  certs on every router that opts in via TLS labels.
- You want **HTTP/3 (QUIC) at the edge** with a
  one-line opt-in: `--entrypoints.websecure.http3=true`
  on a UDP entrypoint enables h3 alongside h2 / h1 with
  Alt-Svc advertised automatically.
- You want **forward-auth as middleware** for a single-
  sign-on shim (Authelia, Authentik, oauth2-proxy):
  one `forwardAuth` middleware attached to multiple
  routers, no per-service auth code.
- You want **a live dashboard** of router / middleware /
  service graph for debugging routing without grepping
  YAML.
- You want **structured logs + Prometheus / OTLP traces
  + access-log JSON** out of the box with
  `--metrics.prometheus=true --tracing.otlp=true`.

## When NOT to use

- You need **the absolute peak L7 throughput / lowest
  latency** at >100k rps per box on commodity HW —
  HAProxy / Envoy / nginx still edge out traefik on
  pure proxy throughput because they spend less time
  on dynamic configuration. Pick HAProxy / Envoy when
  the workload is "millions of static routes, never
  changing".
- You want **a single-binary, file-config-only edge
  with the simplest possible mental model** — pick
  [`caddy`](../caddy/). Caddy's Caddyfile is friendlier
  for "one VPS, three sites, ACME on" than traefik's
  EntryPoint / Provider / Router / Middleware / Service
  decomposition.
- You need **service-mesh data-plane features** (mTLS
  between every pod, L7 policy, outlier detection,
  client-side load balancing with EDS) — that's
  Envoy / Linkerd / Istio territory; traefik is an
  edge router, not a sidecar mesh.
- You need **TCP / UDP load-balancing only with no
  HTTP layer** — HAProxy / IPVS / katran is the
  better tool; traefik can do TCP / UDP routes but
  the design centre is HTTP.
- You need **Windows kernel-level integration** (HTTP.sys,
  IIS shared port) — traefik runs on Windows but is
  a userspace process; pick IIS / YARP if Windows-native
  is the requirement.

## Comparison vs alternatives in zoo

- [`caddy`](../caddy/) — single-binary HTTP server
  with automatic HTTPS as the design centre, Caddyfile
  as the human-friendly config DSL. Pick caddy when
  the topology is static (a few sites, you edit a
  config file) and you want the simplest possible
  ACME story; pick traefik when the topology is
  dynamic (services come and go faster than you
  want to edit a file).
- [`mkcert`](../mkcert/) — local CA for `*.test`-style
  dev certs. Complementary — mkcert seeds the dev box's
  trust store; traefik consumes whatever cert is on
  disk (or fetches from Let's Encrypt in production).
- [`step`](../step/) (if added) — the smallcli ACME
  client + private CA. Complementary — step issues
  certs into a directory; traefik picks them up via
  the file provider.
- [`k9s`](../k9s/) / [`kubectx`](../kubectx/) /
  [`stern`](../stern/) — k8s operator UX. Traefik
  slots in as the ingress controller these UIs route
  *to* in a typical cluster.
- [`nerdctl`](../nerdctl/) / [`podman`](../podman/) —
  container engines. Traefik treats their sockets as
  providers the same way it treats Docker's; the
  label scheme is identical.
- [`consul`](../consul/) / [`nomad`](../nomad/) —
  service discovery + scheduler. Traefik consumes
  Consul's catalog (`--providers.consulcatalog`) and
  Nomad's job tags (`--providers.nomad`) directly,
  so a Nomad job with `tags = ["traefik.enable=true"]`
  becomes routable without a separate ingress layer.

## Why it earns a slot in an AI-native workflow

Most LLM-CLI agent stacks end up shaped like "a handful
of small services": the inference proxy, the embedding
service, an MCP server or two, the vector store,
the eval harness UI, a Jupyter kernel for ad-hoc
analysis, and the chat front-end. They come and go on
a dev box more often than nginx config can keep up
with, they all want TLS in production, and they all
want to be reachable on a pretty hostname rather than
`localhost:7860 / :7861 / :7862 / :11434 / :19530`.
Traefik with the Docker provider turns every
`compose.yaml` service into a labelled route in one
edit (`traefik.http.routers.openwebui.rule=Host('chat.lan')`),
auto-issues certs in production, and gives a
`/dashboard/` view of the live mesh that's much
quicker to read than `docker ps` + `netstat`. For
agent-driven dev environments where containers
spin up in response to a prompt ("scaffold a vector
store + embedding service + chat UI"), an
auto-discovering edge router is the difference
between "everything is reachable on a pretty URL"
and "agent now has to remember a port for each
service it spawned".

## Example invocations

```bash
# Static binary, file provider, two routes, one ACME resolver
traefik \
  --providers.file.filename=/etc/traefik/dynamic.yaml \
  --entrypoints.web.address=:80 \
  --entrypoints.websecure.address=:443 \
  --certificatesresolvers.le.acme.email=admin@example.com \
  --certificatesresolvers.le.acme.storage=/var/lib/traefik/acme.json \
  --certificatesresolvers.le.acme.tlschallenge=true \
  --api.dashboard=true

# Docker provider — labels on the service do the routing
docker run -d --name api \
  -l "traefik.enable=true" \
  -l "traefik.http.routers.api.rule=Host(\`api.example.com\`)" \
  -l "traefik.http.routers.api.tls.certresolver=le" \
  -l "traefik.http.services.api.loadbalancer.server.port=8080" \
  myorg/api:latest

# Compose: edge router + service in one file
cat > compose.yaml <<'YAML'
services:
  edge:
    image: traefik:v3.3
    command:
      - --providers.docker
      - --entrypoints.web.address=:80
    ports: ["80:80"]
    volumes: ["/var/run/docker.sock:/var/run/docker.sock:ro"]
  whoami:
    image: traefik/whoami
    labels:
      - traefik.enable=true
      - traefik.http.routers.whoami.rule=Host(`whoami.localhost`)
YAML
docker compose up -d
curl -H 'Host: whoami.localhost' http://127.0.0.1/

# Kubernetes IngressRoute CRD (richer than core Ingress)
cat <<'YAML' | kubectl apply -f -
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata: { name: api }
spec:
  entryPoints: [websecure]
  routes:
    - match: Host(`api.example.com`)
      kind: Rule
      services:
        - name: api
          port: 8080
      middlewares:
        - name: rate-limit
  tls: { certResolver: le }
YAML

# HTTP/3 on the secure entrypoint
traefik \
  --entrypoints.websecure.address=:443 \
  --entrypoints.websecure.http3=true

# Prometheus metrics + OTLP tracing
traefik \
  --metrics.prometheus=true \
  --tracing.otlp.http.endpoint=http://otel-collector:4318/v1/traces
```
