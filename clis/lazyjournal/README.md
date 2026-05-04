# lazyjournal

> **TUI log viewer for journalctl, syslog files, audit logs,
> Docker / Podman containers, and Kubernetes pods** — a
> single self-contained Go binary that auto-discovers every
> log source the host exposes (systemd units via
> `journalctl`, `/var/log/*` plain-text files,
> `auditctl`/`ausearch`, the Docker / Podman socket, the
> currently configured `kubectl` context) and presents them
> in a left-pane source list + right-pane streaming viewer
> with **live filter-as-you-type** across all sources at
> once — pinned to **0.8.6** (commit
> [`12dee219`](https://github.com/Lifailon/lazyjournal/commit/12dee21921377dbc3b4060fc96078ce8e8408f48),
> [LICENSE](https://github.com/Lifailon/lazyjournal/blob/0.8.6/LICENSE),
> MIT).

Source: <https://github.com/Lifailon/lazyjournal>

## TL;DR

`lazygit` for logs. One TUI keystroke (`Tab` to cycle
panes, `/` to filter) and you go from "is the failure in
the systemd unit, the syslog, the docker container, or
the kube pod?" to "the line is in `nginx.service` at
10:47:13" without remembering the four different incantations
(`journalctl -u nginx -f`, `tail -F /var/log/nginx/error.log`,
`docker logs -f nginx`, `kubectl logs -f deploy/nginx`). The
binary discovers what is on the host and lists it; you arrow-key
through, filter, and go.

The killer property is **unified filter across heterogeneous
log sources**. Type `oom-killer` in the filter bar and every
pane (journal, syslog, container, pod) instantly highlights
matching lines; switch sources with `Tab` and the filter
persists. Six fuzzy / regex / case modes are toggled from one
hotkey. The same workflow that `lazygit` brought to git status
— stop memorizing flags, start arrow-keying — applied to
log triage on a box where the operator does not yet know
*which* log holds the answer.

## Install

```bash
# prebuilt binary (Linux x86_64 / arm64, macOS, FreeBSD)
curl -sSfL \
  https://github.com/Lifailon/lazyjournal/releases/latest/download/lazyjournal-linux-amd64 \
  -o ~/.local/bin/lazyjournal
chmod +x ~/.local/bin/lazyjournal

# Go install
go install github.com/Lifailon/lazyjournal@v0.8.6

# Arch Linux (AUR)
yay -S lazyjournal

# Homebrew
brew install lazyjournal

# Build from source
git clone --branch 0.8.6 https://github.com/Lifailon/lazyjournal
cd lazyjournal
go build -o lazyjournal .
sudo install -m 0755 lazyjournal /usr/local/bin/

# verify
lazyjournal --version    # 0.8.6
```

Single static binary, no runtime dependencies. Reads what
it can with the current user's privileges — run as a user
in the `systemd-journal` / `adm` / `docker` groups (or via
`sudo`) for full coverage of journal + syslog + docker
sources.

## Example usage

```bash
# launch the TUI — auto-discovers all sources
lazyjournal

# filter mode (one shared filter across all sources)
#   /            focus the filter bar
#   <type text>  live-filter all visible log lines
#   Ctrl+w       cycle filter mode (default / fuzzy / regex / timestamp / ...)
#   Esc          clear filter

# pane navigation
#   Tab          cycle: source-list -> filter -> log-view
#   k/j          scroll log view up / down
#   g/G          jump to top / bottom
#   Enter        pin the highlighted source
#   q            quit

# per-source modes
#   journalctl   pick a unit from the list, optionally with --since/--until
#   filesystem   browse /var/log/*, archived .gz files supported transparently
#   docker       list containers, follow stdout/stderr live
#   podman       same as docker via the podman socket
#   kubernetes   list pods in the current context, follow logs live

# CLI flags (most config is in-TUI)
lazyjournal -h                 # full help
lazyjournal --tail 500         # initial buffer size per source
lazyjournal --audit            # include audit.log sources by default
lazyjournal --version
```

Common in-TUI hotkeys:

- `Tab` / `Shift+Tab` cycle panes
- `/` focus the filter bar; `Esc` clear filter
- `Ctrl+w` cycle filter mode (default / fuzzy / regex /
  case-sensitive / timestamp-window / ...)
- `Enter` pin the highlighted source as the active stream
- `Ctrl+s` / `Ctrl+r` jump to next / previous match within
  the log pane
- `g` / `G` jump to top / bottom of the current log
- `?` toggle help overlay
- `q` quit

## Why it matters

- **One TUI for four log surfaces.** Triaging a failed
  deploy on a host that runs systemd + docker + kubelet
  normally means three terminals open with three different
  `-f` commands plus a fourth tail on `/var/log/messages`.
  lazyjournal collapses that into one TUI where the
  operator arrow-keys between sources and the filter
  persists across the switch.
- **Filter mode is the killer feature.** Six filter modes
  (substring, fuzzy, regex, case-sensitive, timestamp-window,
  source-name) cycled with one hotkey — the operator does
  not pre-commit to "this is a regex search" before typing,
  they just type and then refine the mode if too noisy.
  The shared filter across panes means the same query
  `5xx` highlights matching lines in nginx access logs,
  the kube ingress controller pod, and the systemd
  journal entry for the upstream — all in one screen.
- **Discovers what is on the host.** No config file, no
  `--source=...` flags. Run `lazyjournal` on any Linux
  box and the source list reflects what is actually there:
  the systemd units, the `/var/log/*` files (including
  archived `.gz` files which it transparently decompresses),
  the docker containers if the socket is reachable, the
  kube pods if a `kubectl` context is configured. Onboarding
  cost is one binary, not one config-per-environment.
- **Static Go binary.** Drop the binary into a base
  container image, an Alpine sysadmin VM, a NixOS
  installer ISO, or a hardened bastion — no runtime
  dependencies to manage. The same binary works against
  the host's journal *and* the docker socket *and* the
  kubectl context, so a single rescue tool covers all
  three layers.
- **Read-only, side-effect-free.** lazyjournal never
  mutates a unit, restarts a container, or writes to a
  log — purely a viewer/filter. Safe to run on a
  production host during an incident without worrying
  about an accidental keystroke modifying state.

## Vs Already Cataloged

- ai-cli-zoo does not currently catalog any TUI log
  viewer; lazyjournal is the first entry in this niche.
  It composes with [`procs`](../procs/) (per-process
  view) and a future per-process network monitor like
  [`nethogs`](../nethogs/) — procs answers "which
  process is running," nethogs answers "which process
  is on the network," lazyjournal answers "what did
  that process log."
- **Vs `journalctl -fu <unit>` / `tail -F /var/log/...`
  / `docker logs -f` / `kubectl logs -f`:** orthogonal
  on capability but radically different on UX —
  lazyjournal *wraps* all four and adds the pane-switch +
  shared-filter affordances. Use the raw commands in
  scripts and CI; use lazyjournal interactively when the
  operator does not yet know which log holds the
  answer.
- **Vs `lnav`:** overlapping niche, different model.
  `lnav` is a SQL-aware multi-file log navigator with a
  query language and timestamped merge across files —
  the right tool when the operator already has a list of
  log files and wants to *correlate* them with SQL.
  lazyjournal is the right tool when the operator wants
  to *discover* which logs the host even has and
  arrow-key through them. They compose: lazyjournal to
  identify which file/source is interesting, then `lnav`
  on that file for deep query work.
- **Vs `multitail` / `ccze`:** older multi-file
  scrolling viewers. Functional but no journal /
  docker / kube source discovery and no live shared
  filter mode — lazyjournal is the modern answer in the
  same general "one screen, many logs" category.
- **Vs `stern` / `kubetail`:** `stern` is the
  best-in-class kube-pod log multiplexer with regex
  pod-name selection and per-line color coding.
  lazyjournal includes kube logs as one of four
  sources but does not have stern's pod-selector
  ergonomics — for kube-only workflows on a kube-only
  host, prefer `stern`. For mixed
  systemd+docker+kube+syslog hosts, lazyjournal is the
  unifier.
- **Vs Grafana Loki / Elastic Kibana:** the
  long-running observability stack indexes logs across
  the fleet and supports historical querying.
  lazyjournal is the single-host, ten-second answer
  when the dashboard is not yet wired up, the box is
  not exporting logs, or the question is "what is
  happening *right now* on *this* host."

## License

MIT — see
[LICENSE](https://github.com/Lifailon/lazyjournal/blob/0.8.6/LICENSE).
Static-link the binary into closed-source distributions,
ship it inside container images, vendor it into a NixOS
module — all unrestricted under MIT terms.

## Caveats

- **Linux is the primary target.** The journalctl /
  `/var/log` / audit sources are Linux-specific; the
  docker / podman / kubernetes sources work on macOS too
  via the docker-for-mac socket and a configured
  `kubectl` context. macOS users get a useful subset
  (filesystem + docker + kube) but no systemd journal.
- **Privileges shape the source list.** A user not in
  `systemd-journal` / `adm` / `docker` groups will see
  an incomplete source list (only world-readable
  `/var/log/*` files plus their own units). Either add
  the user to those groups or run via `sudo`.
- **Kubernetes uses the active `kubectl` context.**
  Switching contexts in another shell does not
  retroactively update a running lazyjournal — restart
  the TUI after a `kubectl config use-context` change.
- **Filter buffers, not files.** The filter applies to
  the in-memory buffer (`--tail` size); raise `--tail`
  to a higher value if filtering needs to reach further
  back in the journal/file. For deep historical
  queries, hand off to `journalctl --since=...` /
  `lnav` / Loki.
- **Active 0.x line — the CLI surface is stable but
  hotkeys may shift across minor versions.** Pin to a
  specific version in shared dotfiles / images
  (`go install github.com/Lifailon/lazyjournal@0.8.6`)
  if reproducibility matters. The 0.8.x line has been
  stable across recent releases.
- **Read-only by design.** lazyjournal cannot
  `systemctl restart`, `docker restart`, or
  `kubectl delete pod` — and intentionally so. Pair
  with [`k9s`](https://k9scli.io/) or `systemctl` /
  `docker` directly for mutating actions.

## As of

2026-05-04. Upstream tag `0.8.6` (2026-03-16). Active 0.x
release line (≈ monthly cadence); the four-source discovery
model and the shared-filter UX are stable across recent
releases. Re-verify on a future 1.0 release that may reshape
the source-discovery API.
