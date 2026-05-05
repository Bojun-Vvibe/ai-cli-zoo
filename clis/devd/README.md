# devd

> **A local web server for developers** — a single Go binary
> that serves a directory over HTTP/HTTPS with auto-generated
> certs, livereload-on-file-change, reverse proxying, request
> logging, throttling, and latency injection. Pinned to **v0.9**
> (commit `689756b32ac190d75ab64d799c7428939504647a`,
> [LICENSE](https://github.com/cortesi/devd/blob/master/LICENSE),
> MIT).

Source: <https://github.com/cortesi/devd>

## TL;DR

`devd` is what you reach for when you need an HTTP server in front
of a directory or upstream service for the next thirty minutes —
demoing a static site, exercising a frontend against a mocked
API, sharing a single page with a colleague over a tunnel, or
debugging exactly what request a misbehaving client is sending.
One binary, no config file required, sane defaults: serves the
current directory on a random port, prints every request to
stdout in colourised single-line form, generates a self-signed
TLS cert on demand, and watches the served tree to reload
connected browsers when a file changes.

## Install

```bash
# Homebrew (macOS / Linux)
brew install devd

# Go install
go install github.com/cortesi/devd/cmd/devd@latest

# from a release tarball (any OS)
curl -Lo devd.tgz https://github.com/cortesi/devd/releases/download/v0.9/devd-0.9-linux64.tgz
tar xf devd.tgz
sudo install devd-0.9-linux64/devd /usr/local/bin/

# verify
devd --version    # devd 0.9
```

## License

MIT — see
[LICENSE](https://github.com/cortesi/devd/blob/master/LICENSE).

## One Concrete Example

```bash
# 1. serve current directory on a random port, open the URL,
#    log every request, livereload on file changes
devd -ol .

# 2. serve over HTTPS with an auto-generated self-signed cert
devd -S .

# 3. reverse-proxy /api/* to a backend on localhost:8080,
#    and serve static assets from ./dist on /
devd -w ./dist /api=http://localhost:8080 ./dist

# 4. inject 200 ms latency and cap throughput at 50 KB/s
#    to simulate a slow mobile connection
devd -d 200 -u 50 .

# 5. mount the same directory at multiple paths
devd /static=./dist /api=http://localhost:8080 ./dist

# 6. live-reload only HTML/CSS/JS, ignore node_modules
devd -w '**/*.{html,css,js}' --excludes node_modules .

# 7. emit access log as JSON for piping
devd --logtime --logheaders . 2>&1 | jq -R 'fromjson? // .'

# 8. share a directory over an ngrok-style tunnel
devd -A 0.0.0.0 -P 8080 .   # then ngrok / cloudflared / sshx the port
```

## Niche It Fills

**HTTP development server with sharp edges.** The default
candidates for "serve this directory" — `python -m http.server`,
`npx serve`, `caddy file-server` — each cover the basic case but
miss the development-specific verbs: livereload on file change
without a build-tool plugin, reverse proxy a real backend behind
a static frontend without writing a `Caddyfile`, inject latency
and throttle bandwidth to reproduce a slow-connection bug,
auto-issue a self-signed cert so the page testing
`navigator.geolocation` (HTTPS-only API) works on `127.0.0.1`,
log requests with timing in a parseable single line. `devd` is
the one binary that does all of those out of the box, no plugin,
no config file.

## Why use it

1. **Reverse proxy + static serve in one process.** Mounting
   `./dist` on `/` and `http://localhost:8080` on `/api` is two
   command-line arguments, not a 30-line nginx config. CORS
   "works" because the browser sees one origin. Frontend
   development against a backend running in another shell becomes
   one terminal and one open browser tab.
2. **Livereload without a build-tool plugin.** `devd -L` watches
   the served tree and injects a tiny WebSocket client into HTML
   responses; on file change the connected browser reloads. Works
   for vanilla HTML/CSS/JS without webpack/vite/parcel — the
   right tool when you're prototyping a one-file page or
   debugging a static-export bundle from someone else's build.
3. **Network-condition simulation built in.** `-d` (downstream
   latency) and `-u` (upstream throttle) reproduce slow-mobile
   bugs without `tc qdisc`, Network Link Conditioner, or Chrome
   DevTools' "Slow 3G" preset (which only throttles in DevTools,
   not from another tab). Combined with `-S` (auto HTTPS), it's
   the closest single-command approximation of a real flaky
   mobile network on `localhost`.

For an LLM-CLI agent that generates a static site or a frontend
build, `devd <dir>` is a one-line "serve this so the user can
preview" tool call. Auto-issued cert means HTTPS-only browser
APIs work in the preview.

## Vs Already Cataloged

- **Vs [`miniserve`](../miniserve/):** miniserve is the
  *file-sharing* shape — directory-listing UI, upload form,
  QR-code-the-URL, archive-on-the-fly download. devd is the
  *web-development* shape — reverse proxy, livereload, latency
  injection, self-signed TLS, request logging. Pick miniserve to
  hand a colleague a directory of files; pick devd to develop a
  page against a backend.
- **Vs [`caddy`](../caddy/) `file-server` / `reverse_proxy`:**
  Caddy is the production-grade web server (auto-cert from
  Let's Encrypt for real domains, HTTP/3, layered access control,
  modular plugins). devd is the dev-server slice of Caddy's
  feature set with a one-line CLI instead of a `Caddyfile`,
  optimised for "throw away in 30 minutes". Pick Caddy to actually
  host the production site; pick devd to iterate on it.
- **Vs [`mitmproxy`](../mitmproxy/) for request inspection:**
  mitmproxy is the deep request-modification tool (rewrite headers,
  replay flows, MITM TLS for arbitrary clients). devd's logging
  is single-line stdout — fine for "what did the page request"
  visibility, not for editing the response. Compose: devd hosts
  the page, mitmproxy upstream of devd captures the API traffic
  at L7.
- **Vs `python -m http.server` / `npx serve`:** devd is `serve`
  with the sharp edges (proxy, TLS, livereload, throttling,
  logging) built in. Pick the Python/Node one-liner when you have
  no choice (no Go binary handy); pick devd for everything else.

## Caveats

- **Slow release cadence.** v0.9 was tagged in 2020 and the
  project is on intermittent maintenance. The author's
  attention shifted to other tooling. It still works against
  current Go runtime; pin the version in install scripts.
- **Auto-cert is self-signed.** Browsers warn on first
  connection — click through once per host. Useful for
  `localhost` development; the wrong tool for any host the
  public will reach (use Caddy for production TLS).
- **Livereload injects into HTML responses.** A page that
  returns `application/octet-stream` or strict CSP without
  `'unsafe-inline'` won't get the injected script. Disable
  with `-L=false` if it interferes with what you're testing.
- **No graceful upgrade / hot reload.** Reload via
  `Ctrl-C` + restart. Restart cost is ~10 ms (Go binary, no
  certificate regeneration unless `-S` cert file was deleted),
  so this is rarely felt.
- **Logging is human-shaped by default.** For machine parsing,
  `--logtime --logheaders` together produce a more uniform line
  but the format is still bespoke devd, not Apache common-log
  or JSON. Pipe through `awk` or `cut` for ad-hoc analysis;
  prefer Caddy's structured JSON log if your workflow consumes
  logs programmatically.
- **No HTTP/2, no HTTP/3.** HTTPS is TLS-over-HTTP/1.1. For
  frontends that depend on H2 multiplexing in the dev loop,
  pair devd with a Caddy reverse proxy in front, or skip devd.
