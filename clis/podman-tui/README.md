# podman-tui

> **Terminal UI for managing local Podman containers, pods,
> images, volumes, and networks** — k9s-style navigation but for
> the rootless container engine. Single Go binary that talks to
> the Podman REST API over the user's UNIX socket, no Docker
> daemon required. Pinned to **v1.11.1** (SPDX: `Apache-2.0`,
> [LICENSE](https://github.com/containers/podman-tui/blob/main/LICENSE)).

Source: <https://github.com/containers/podman-tui>

## TL;DR

`podman-tui` gives the Podman ecosystem the same "look at
everything running, kill the wrong thing fast" interactive
surface that `lazydocker` and `k9s` give Docker and Kubernetes.
You launch it, it connects to your local (or remote) Podman
socket, and you get tabs for containers, pods, images, volumes,
and networks. Arrow-key driven, single-key actions for
`start/stop/kill/rm/exec/logs`, no YAML, no daemon to manage.

## Install

```bash
# Homebrew (macOS / Linux)
brew install podman-tui

# Pre-built release binary
curl -Lo podman-tui "https://github.com/containers/podman-tui/releases/download/v1.11.1/podman-tui-v1.11.1-linux-amd64"
chmod +x podman-tui
sudo mv podman-tui /usr/local/bin/

# From source
git clone https://github.com/containers/podman-tui
cd podman-tui && make binary

# verify
podman-tui --version    # podman-tui version 1.11.1
```

Requires a running Podman socket: `systemctl --user start podman.socket`
on Linux, or `podman machine start` on macOS/Windows.

## License

Apache-2.0 — see
[LICENSE](https://github.com/containers/podman-tui/blob/main/LICENSE).

## One Concrete Example

```bash
# 1. start the user-level podman socket (Linux)
systemctl --user start podman.socket

# 2. launch the TUI
podman-tui

# 3. inside the TUI:
#    - Tab / Shift-Tab : cycle Containers / Pods / Images / Volumes / Networks / System
#    - arrows          : pick a row
#    - Enter           : open detail / inspect
#    - s / k / r       : start / kill / remove the highlighted resource
#    - l               : tail logs
#    - e               : exec into a shell
#    - /               : filter the current view

# 4. point at a remote podman socket
podman-tui --url unix:///run/user/1000/podman/podman.sock
podman-tui --url ssh://core@vm.local/run/user/1000/podman/podman.sock

# 5. dump the in-memory config the TUI is using
podman-tui --config-path ~/.config/podman-tui/podman-tui.json
```

## Niche It Fills

**Interactive operator surface for rootless Podman.** If you
already use Podman as a Docker drop-in but miss the "one screen
shows me everything" view, this is it. Especially useful on
single-host servers and developer laptops where you have ten
short-lived containers and need to find the one that is wedged.

## Why use it

1. **Native Podman, not "Docker-compatible".** Talks to the
   Podman REST API directly, so it surfaces pods (a Podman
   primitive Docker has no equivalent for) as a first-class
   tab.
2. **Rootless-first.** Connects to `$XDG_RUNTIME_DIR/podman/podman.sock`
   by default — no `sudo`, no daemon, no group membership games.
3. **Remote sockets via SSH.** Manage a Podman host from your
   laptop with `--url ssh://...`, same TUI, same keys.
4. **Single static binary.** Drop on a server, run, exit. No
   Python runtime, no Electron, no browser.
5. **Same mental model as `k9s` / `lazydocker`.** Tabs across
   resource kinds, single-key verbs, `/` to filter. No new
   muscle memory if you already use either.

## Vs Already Cataloged

- **Vs [`lazydocker`](../lazydocker/):** Same UX category,
  different engine. `lazydocker` requires the Docker daemon;
  `podman-tui` requires only the user-level Podman socket.
- **Vs [`k9s`](../k9s/):** `k9s` is for Kubernetes clusters.
  `podman-tui` is for the local container engine. Use both.
- **Vs [`ctop`](../ctop/):** `ctop` is read-mostly resource
  metrics. `podman-tui` is fully interactive (start, stop, exec,
  delete). Different intent.
- **Vs [`oxker`](../oxker/) / [`dozzle`](../dozzle/):** Those
  target Docker. `podman-tui` is the Podman counterpart.

## Caveats

- **Needs the Podman socket up.** First-run failures are almost
  always `systemctl --user start podman.socket` or
  `podman machine start` on Mac.
- **Not a Compose UI.** Compose-style stacks live in
  `podman compose` or `quadlet`; `podman-tui` shows the
  resulting containers/pods, not the compose file.
- **TUI only.** No headless / scriptable mode — for CI, drive
  `podman` itself.
- **Socket auth = whoever can read the socket.** Treat
  `--url ssh://...` like an SSH session: anyone with that
  socket has full container control.
