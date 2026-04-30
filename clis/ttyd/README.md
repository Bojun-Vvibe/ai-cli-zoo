# ttyd

- **Repo:** https://github.com/tsl0922/ttyd
- **Latest version:** 1.7.7 (released 2025-03-30)
- **License:** MIT — SPDX `MIT` — [LICENSE](https://github.com/tsl0922/ttyd/blob/main/LICENSE)
- **Category:** share a terminal over the web (xterm.js front-end + libwebsockets back-end)

A **single C binary that turns any local command (default: `bash`)
into a browser-accessible terminal** served over WebSocket, with the
xterm.js front-end bundled in. Run `ttyd bash` and a fully-featured
terminal — colors, mouse, resize, copy/paste, IME, file
upload/download, reconnect-on-disconnect — is reachable at
`http://localhost:7681/`. The binary is self-contained (libwebsockets
+ json-c + libuv statically linked), the protocol is a simple
WebSocket frame format, and the same one binary serves the static
xterm.js assets, so deploying a web-shell to a fresh Linux box is
literally `scp ttyd host: && ssh host './ttyd bash'`. Auth (basic or
custom-header), TLS (built-in or behind a reverse proxy), read-only
mode, max-clients caps, custom terminal commands per-route, and
arbitrary client args (`ttyd htop`, `ttyd vim file.txt`, `ttyd tmux
attach`) are all flags.

## Why it's interesting

For an LLM-CLI-driven workflow, `ttyd` is the **simplest path from
"a long-running CLI agent on a remote box" to "a shareable URL a
collaborator can attach to from a phone."** Wrap an `aider` /
`codex` / `claude` session in `ttyd -W aider` and the read-write
session is one URL away; flip `-R` for read-only and the same URL
becomes a live demo a reviewer can follow without keyboard access.
Unlike SSH-based sharing tools, the client is just a browser — no
cert install, no port-forward, no client binary, works through
corporate captive portals that allow HTTPS but not arbitrary TCP.
1.7.7 fixes version detection when not building from a git checkout.

## Install

```bash
# macOS
brew install ttyd

# Debian / Ubuntu (universe)
sudo apt-get install -y ttyd

# Arch
sudo pacman -S ttyd

# Static binary from a release
curl -L -o ttyd https://github.com/tsl0922/ttyd/releases/download/1.7.7/ttyd.x86_64
chmod +x ttyd && sudo mv ttyd /usr/local/bin/
```

## Sample invocation

```bash
# Read-write bash on :7681
ttyd -W bash

# Read-only htop on :8080 with basic auth
ttyd -p 8080 -R -c admin:secret htop

# Share a tmux session over TLS, max 5 viewers
ttyd -p 443 \
     -C /etc/ssl/cert.pem -K /etc/ssl/key.pem \
     -m 5 \
     tmux new -A -s shared
```

## Comparable to

- [`tmate`](../tmate/) — SSH-relay-based pairing; needs an SSH
  client on the viewer side (`ttyd` only needs a browser).
- `gotty` — older Go-based predecessor in the same niche; largely
  unmaintained, which is why `ttyd` has become the default.
