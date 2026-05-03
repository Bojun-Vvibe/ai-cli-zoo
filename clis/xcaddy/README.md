# xcaddy

- **Repo:** https://github.com/caddyserver/xcaddy
- **Version:** v0.4.5 (latest stable)
- **License:** Apache-2.0 ([LICENSE](https://github.com/caddyserver/xcaddy/blob/master/LICENSE))
- **Language:** Go
- **Install:** `brew install xcaddy` · `apt install xcaddy` (Cloudsmith repo per upstream docs) · `go install github.com/caddyserver/xcaddy/cmd/xcaddy@latest` · static binaries on the GitHub release page (`xcaddy_0.4.5_mac_arm64.tar.gz`, `xcaddy_0.4.5_linux_amd64.tar.gz`, `xcaddy_0.4.5_windows_amd64.zip`)

## What it does

`xcaddy` is the official custom-build tool for the [`caddy`](../caddy/)
web server. Stock `caddy` ships a fixed set of standard modules; the
moment you need a non-default DNS provider for ACME-DNS challenges
(Cloudflare, Route53, DigitalOcean, Hetzner, Gandi, ~70 more), a
storage backend other than the local filesystem (Redis, S3, Postgres,
Consul, etcd), an auth/rate-limit/transform module from the community,
or an in-house Go plugin, you have to recompile `caddy` with those
modules linked in. `xcaddy build` does exactly that: `xcaddy build
v2.10.0 --with github.com/caddy-dns/cloudflare --with
github.com/mholt/caddy-ratelimit --with example.com/me/mymodule` reads
the requested Caddy version, fetches each `--with` module (any Go
module path works, with optional `@version` and optional `=local/path`
override for local development), generates a tiny `main.go` that
imports them all, runs `go build`, and drops a single static `caddy`
binary in the current directory that has those modules baked in. The
second mode, `xcaddy run`, is the development loop for writing your
own Caddy module: from inside your module's source tree, `xcaddy run
--config Caddyfile` builds a temporary Caddy that imports your local
package via a Go module replace directive, runs it under your
`Caddyfile`, and re-builds + restarts on file change. The whole tool
is one static Go binary, no Docker or build server required, and the
output binary is identical in shape to a stock `caddy` download — same
admin API, same Caddyfile, same `caddy reload` semantics.

## When to pick it / when not to

Pick `xcaddy` whenever your `caddy` deployment needs a module the
stock binary does not include — which is almost any production deploy
that does ACME-DNS for wildcard certificates, since the DNS-provider
modules live outside the core. The build is fully reproducible
(`xcaddy build v2.10.0 --with github.com/caddy-dns/cloudflare@v0.2.1`
pins both Caddy and the module), takes 20–60 seconds on a laptop, and
the output is a single ~50 MB static binary you can `scp` straight to
a server or copy into a `FROM scratch` Docker image. For module
authors, `xcaddy run` is the dev loop you want: edit Go, save, the
running Caddy hot-restarts with your changes loaded, no manual
`go build` cycle, no `replace` directive surgery in `go.mod`. Skip
`xcaddy` if you only need stock Caddy (just `brew install caddy`), if
you ship Caddy via a distro package and accept whatever modules that
package was built with, or if you have already adopted a more general
build system (Bazel, Earthly, a CI image with `go build` baked in)
that handles the same plugin-graph problem for multiple Go binaries
at once. The only meaningful production caveat: `xcaddy` does not
verify module signatures beyond what `go mod` itself does — pin
versions in CI, mirror modules through your own proxy
(`GOPROXY=https://your-proxy`) if you care about supply chain.

## Vs already cataloged

- **Vs [`caddy`](../caddy/):** `caddy` is the server; `xcaddy` is
  what you use to build a non-default `caddy`. They are paired, not
  alternatives. Stock `caddy` covers the "TLS-terminating reverse
  proxy in front of an app on the same host" case completely; `xcaddy`
  unlocks every external module the Caddy ecosystem ships
  (DNS providers, storage backends, custom auth, custom transports,
  in-house Go plugins).
- **Vs [`buildah`](../buildah/) / [`apko`](../apko/) / [`ko`](../ko/):**
  orthogonal. Those are general-purpose container/image builders.
  `xcaddy` produces a plain Go binary; you would use `ko` or `apko`
  to wrap that binary in a minimal image for distribution.

## Caveats

- The output binary is **only as reproducible as your `go.sum` and
  `GOPROXY`**. For audit-grade reproducibility, pin every `--with`
  module to an exact version, set `GOFLAGS=-trimpath`, and build
  inside a fixed Go toolchain version (e.g. via
  [`mise`](../mise/) or a CI image).
- `xcaddy` requires a Go toolchain on the build host (Go 1.22+ for
  current Caddy versions). It is not a binary-only solution; the
  build step is real.
- `xcaddy run` development loop assumes your module's `go.mod`
  declares a Caddy dependency — the replace it injects is for *your*
  module, not for Caddy itself. For Caddy-core hacking, build Caddy
  from source directly.
- Apache-2.0 ([LICENSE](https://github.com/caddyserver/xcaddy/blob/master/LICENSE))
  — permissive; safe to ship the `xcaddy` binary itself in CI images
  and the resulting `caddy` binary in production. Each `--with` module
  carries its own license; verify before linking proprietary modules.
