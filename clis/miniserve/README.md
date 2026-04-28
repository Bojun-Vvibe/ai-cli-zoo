# miniserve

- **Repo:** https://github.com/svenstaro/miniserve
- **Version:** v0.35.0 (latest stable, 2026)
- **License:** MIT ([LICENSE](https://github.com/svenstaro/miniserve/blob/master/LICENSE))
- **Language:** Rust
- **Install:** `brew install miniserve` · `cargo install miniserve` · `pacman -S miniserve` · `apt install miniserve` (Debian/Ubuntu) · prebuilt binaries on the GitHub release page · binary name is `miniserve`

## What it does

`miniserve` is a single static Rust binary that serves a directory
(or a single file) over HTTP with sensible defaults: nicely
formatted directory listings, optional QR code of the URL printed
to the terminal on start, optional basic auth, optional TLS, and
optional upload form. It binds `0.0.0.0:8080` by default, prints
every request as a colored access log line, and exits cleanly on
Ctrl-C. Zero config file, no dependencies, no runtime, no
framework — `miniserve .` and the directory is reachable from
every device on the LAN in under a second.

It also speaks `tar.gz`, `tar`, and `zip` archive-on-demand
(`?download=tar.gz` on a directory streams a compressed archive of
the subtree), serves a single file at the root with
`miniserve ./file.pdf`, and supports `--upload-files` to accept
POSTed uploads back into the served directory.

## When to pick it / when not to

Reach for `miniserve` when you need to hand a directory or a file
to another device — your phone, a teammate's laptop on the same
WiFi, a freshly-provisioned server that needs a tarball — and you
do not want to spin up `python3 -m http.server` (slower, no auth,
no upload, no zip-on-demand) or stand up `nginx`. It is also the
right tool for showing a static site to a coworker over Tailscale
or a temporary tunnel, dropping a build artifact onto a phone via
the QR code, or accepting a one-shot upload from a device that
cannot ssh.

Skip it for production hosting — there is no caching layer, no
HTTP/2 push, no virtual hosts; reach for `caddy` or `nginx`. Skip
it if you need fancy auth (OIDC, mTLS) — basic auth is the only
option. Skip it if you need WebDAV semantics rather than HTTP
GET/POST.

## Why it matters in an AI-native workflow

Coding agents that run inside sandboxes
([`container-use`](../container-use/), E2B, Codespaces, devcontainers)
frequently produce artifacts a human needs to *look at*: a
generated PDF report, a built static site, a `pprof` SVG flame
graph, a tarball of N files for review. Pulling those out via
`scp`/`docker cp` is friction, and a hosted file-share is overkill
for a 30-second handoff. `miniserve <artifact-dir>` running inside
the sandbox plus a port-forward (or a Tailscale-attached sandbox)
gives the human a clickable URL with a directory listing, instant
zip-of-the-whole-thing, and access logs the agent can grep to
confirm "the human actually downloaded it." It is also the
canonical way to expose `code2prompt` / `repomix` / `gitingest`
output to a model running on a different machine without standing
up real infrastructure.

## Example invocations

```bash
# Serve the current directory on 0.0.0.0:8080 with a directory listing
miniserve .

# Serve a single file (the URL points to the file itself, not a listing)
miniserve ./build.tar.gz

# Bind to a specific interface and port, print a QR code of the LAN URL
miniserve --interfaces 192.168.1.42 --port 9000 --qrcode .

# Require basic auth (user:password or user:sha256:hash)
miniserve --auth admin:s3cret .

# Allow upload from the served page's form (writes to the served dir)
miniserve --upload-files .

# Enable on-demand archive download (browser gets ?download=tar.gz button)
miniserve --enable-tar-gz --enable-zip .

# Serve over HTTPS with a self-signed cert (or pass --tls-cert / --tls-key)
miniserve --tls-cert ./cert.pem --tls-key ./key.pem .

# Hide hidden files (.git, .env) from the listing
miniserve --hidden=false .

# Custom index file (acts like nginx try_files index.html)
miniserve --index index.html .
```

## Alternatives in this catalog

- [`rclone`](../rclone/) — `rclone serve http` does the same job
  with cloud-backend mounts; pick `rclone serve` if the bytes live
  in S3/GCS, pick `miniserve` for local files.
- [`croc`](../croc/) — peer-to-peer one-shot file transfer with
  end-to-end encryption; pick `croc` when the receiver is *not* on
  the same LAN, pick `miniserve` when they are and you want a
  browser, not a CLI, on the receiver side.
- [`monolith`](../monolith/) — packages a *remote* webpage into a
  single self-contained HTML file you might then `miniserve`.
