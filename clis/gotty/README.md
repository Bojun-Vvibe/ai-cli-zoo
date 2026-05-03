# gotty

> **A small Go program that turns any CLI command
> into a read-only (or read/write) web app over
> WebSocket** — `gotty htop`, `gotty -w bash`,
> `gotty tmux attach` — and the program runs on
> the server while a browser tab on someone else's
> machine sees the live `xterm.js`-rendered TTY,
> resizes correctly, and reconnects automatically.
> The maintained community fork by `sorenisanerd`
> tracks Go-stdlib changes and ships fresh release
> binaries. Pinned to **v1.6.0**
> ([LICENSE](https://github.com/sorenisanerd/gotty/blob/master/LICENSE),
> MIT).

Source: <https://github.com/sorenisanerd/gotty>

## TL;DR

You're SSH'd into a host. You want to *show*
someone — a coworker, a contractor on a personal
device, your own laptop on hotel Wi-Fi — what
`htop` / `tail -F` / a long-running `make` looks
like *right now* on this box. SSH-ing them in
needs an account, a key, a port, a jump host, a
firewall rule. Screen-sharing needs a meeting tool.
Pasting a screenshot loses the live update.
`gotty` turns the program into a URL: `gotty htop`
on the server prints
`http://server:8080/`, the viewer opens it in any
browser and watches the same TTY in real time
through `xterm.js`. Read-only by default; opt into
input with `-w`. The program receiving the
keystrokes is the one you launched — `gotty bash`
gives a shell, `gotty tmux a -t demo` joins a
session, `gotty top` gives a viewer.

## Install

```bash
# Homebrew tap (community fork)
brew install sorenisanerd/tap/gotty

# Pre-built binaries from the release page
curl -L -o /tmp/gotty.tgz \
  https://github.com/sorenisanerd/gotty/releases/download/v1.6.0/gotty_v1.6.0_linux_amd64.tar.gz
tar -xzf /tmp/gotty.tgz -C /usr/local/bin gotty
chmod +x /usr/local/bin/gotty

# Go install
go install github.com/sorenisanerd/gotty@v1.6.0

# Verify
gotty --version    # GoTTY 1.6.0
```

Single static Go binary, zero runtime. Drop into
a sidecar container, a Dockerfile `RUN`, an
initramfs, or `~/bin` and it works.

## Use it for

```bash
# 1) Read-only TTY broadcast (default)
gotty htop
#   → 2026/01/01 12:00:00 GoTTY is starting...
#   → URL: http://0.0.0.0:8080/

# Anyone on the network can open the URL and *watch* htop;
# they cannot send keystrokes.

# 2) Read/write — viewer can type
gotty -w bash
# Be deliberate. -w on a shell is full RCE for whoever
# loads the page. Pair with --credential and --port.

# 3) Basic-auth protect the URL
gotty -w --credential alice:s3cret tmux a -t demo

# 4) Custom port + bind address (default is :8080)
gotty -p 9000 -a 127.0.0.1 htop
# 127.0.0.1 means "tunnel to it via ssh -L 9000:localhost:9000"

# 5) Permanent share (kept alive across reconnects)
gotty -w --reconnect --reconnect-time 30 tmux new-session -A -s share

# 6) TLS in-process
gotty --tls --tls-crt server.crt --tls-key server.key htop

# 7) Run inside a container as a sidecar exposing logs
docker run --rm -p 8080:8080 -v /var/log:/var/log:ro \
  ghcr.io/sorenisanerd/gotty:v1.6.0 \
  tail -F /var/log/syslog

# 8) Pass HTTP headers as environment variables to the program
gotty --pass-headers '*' my-handler.sh
# Useful behind an auth proxy that injects X-User / X-Token
```

The full flag set is small: `-w`, `-p`, `-a`,
`--credential`, `--reconnect`, `--reconnect-time`,
`--tls`, `--max-connection`, `--once`,
`--permit-arguments` (allow viewer to append CLI
args via query string), `--pass-headers`. Config
can also live in `~/.gotty` (HCL).

## Why include it in a CLI catalog

1. **It is the canonical "command-line as a URL"
   tool.** No chat overlay, no custom client, no
   account system — just a program → an HTTP URL
   → an `xterm.js` view. For demoing a long-
   running task, broadcasting a deploy, watching a
   fleet of `htop`s on different hosts, or
   exposing a TUI to a teammate with no SSH access,
   `gotty` is the smallest piece of plumbing that
   does it.
2. **Single static Go binary, zero runtime.**
   Drops into Alpine / distroless / scratch
   images, into a sidecar container, into a
   release-engineering Jumpbox, into an embedded
   device with a few MB free. The on-disk
   footprint is small enough to bake into golden
   images.
3. **The community fork is alive.** The original
   `yudai/gotty` was archived in 2017;
   `sorenisanerd/gotty` keeps it building against
   modern Go, ships releases for darwin-arm64,
   freebsd, openbsd, solaris, and continues to
   accept fixes. Picking the maintained fork is
   the difference between "abandoned in 2017" and
   "released August 2025".

For an LLM-CLI workflow, `gotty` is the "expose
the agent's terminal session to a human reviewer
in real time" piece: spin up the agent inside
`gotty -w tmux a`, the operator opens the URL and
watches every command the agent runs, ready to
take over the keyboard if it goes off-script. Pair
with [`asciinema`](../asciinema/) for *recorded*
sessions; `gotty` is the *live* counterpart.

## Vs Already Cataloged

- **Vs [`asciinema`](../asciinema/):** different
  axis. `asciinema` *records* and uploads a TTY
  session for asynchronous playback later;
  `gotty` *broadcasts* a live session right now.
  Same `xterm.js` rendering on the viewer side,
  totally different temporal model.
- **Vs [`ttyd`](../ttyd/):** direct sibling — both
  expose a TTY over WebSocket. `ttyd` is C with
  `libwebsockets`, more emphasis on full
  read/write shells, OAuth, and Argo / Kubernetes
  pod-shell deployments. `gotty` is Go, smaller,
  more emphasis on read-only broadcast and the
  "command-as-URL" mental model. Pick by language
  and integration story.
- **Vs `tmate`:** `tmate` is for *peer-to-peer
  shell sharing* over a hosted relay (you give
  someone an `ssh` URL). `gotty` is for *web
  browser* sharing without an SSH client. Picky
  audience can use `tmate`; everyone-with-a-
  browser audience uses `gotty`.
- **Vs Jupyter / web IDEs:** orthogonal — those
  expose a *kernel* / *editor*. `gotty` exposes a
  raw TTY running whatever program you specify.

## Caveats

- **`-w` is full remote command execution for
  whoever loads the URL.** Always pair with
  `--credential user:pass`, bind to `127.0.0.1`
  + SSH-tunnel, or front with an auth proxy. The
  README is explicit; treat it like exposing a
  shell, because that's what it is.
- **Basic auth + plain HTTP is the default.**
  Enable `--tls` with a real cert, or terminate
  TLS at a reverse proxy (nginx / caddy /
  traefik). Don't ship credentials over plain
  HTTP across an untrusted network.
- **`xterm.js` is the only frontend.** Modern
  browsers handle it fine; very old browsers or
  limited embedded webviews may not render
  certain Unicode / colour modes.
- **One viewer per process by default.**
  `--max-connection N` lifts the cap, but each
  viewer shares the same TTY — there is no
  per-viewer multiplexing. For independent
  sessions, run multiple `gotty` processes on
  different ports or wrap with `tmux`.
- **MIT license** — permissive; safe to ship
  inside proprietary container images and on-prem
  appliances with attribution.
