# bandwhich

> **Terminal bandwidth utilisation by process, connection, and
> remote host** — a Rust TUI that sniffs network traffic on a
> chosen interface (libpcap on macOS / Linux, npcap on Windows)
> and renders three live-updating tables: which **processes** are
> using bandwidth right now, which **remote hosts** they are
> talking to (with reverse-DNS), and which **connections**
> (5-tuples) carry the most bytes — refreshed every second with
> per-row up/down rates and totals. Pinned to **v0.23.1** (commit
> `adc6aa26aa25b448374d92941582057a550d4e5b`,
> [LICENSE.md](https://github.com/imsnif/bandwhich/blob/main/LICENSE.md),
> MIT).

Source: <https://github.com/imsnif/bandwhich>

## TL;DR

`bandwhich` is what you reach for when `nethogs` shows you
"something is uploading 30 MB/s" but not *what* and not *to
where*. The three-pane TUI joins kernel-side socket → PID
mappings with packet capture, so you immediately see "Slack is
sending 12 MB/s to `edgeapi.slack.com`," "`docker` is pulling 80
MB/s from `cloudflare.com`," and "`ssh` is steady at 4 KB/s to
`bastion.internal` over `tcp/22`," all in one screen. No
configuration; one command, one screen, one answer.

## Install

```bash
# Homebrew (macOS / Linux)
brew install bandwhich

# Cargo
cargo install --locked bandwhich

# Linux package managers
# Arch:    pacman -S bandwhich
# Debian/Ubuntu: download .deb from releases page
# Fedora:  dnf install bandwhich
# Nix:     nix-env -iA nixpkgs.bandwhich
# Alpine:  apk add bandwhich

# from a release tarball (any OS)
curl -Lo bandwhich.tar.gz "https://github.com/imsnif/bandwhich/releases/download/v0.23.1/bandwhich-v0.23.1-aarch64-apple-darwin.tar.gz"
tar xf bandwhich.tar.gz
sudo install bandwhich /usr/local/bin/

# verify
bandwhich --version    # bandwhich 0.23.1
```

`bandwhich` requires raw-packet privileges. On Linux either run
under `sudo` or grant once with
`sudo setcap cap_net_raw,cap_net_admin=eip $(which bandwhich)`.
On macOS run with `sudo` (requires access to the BPF devices).

## License

MIT — see [LICENSE.md](https://github.com/imsnif/bandwhich/blob/main/LICENSE.md).
Permissive, no attribution required for binaries.

## One Concrete Example

```bash
# 1. live TUI on the default interface (auto-detected)
sudo bandwhich

# 2. point at a specific interface (Wi-Fi on macOS, eth0 on Linux)
sudo bandwhich -i en0
sudo bandwhich -i eth0

# 3. raw mode: stream connection lines to stdout (great for tee/grep)
sudo bandwhich --raw | grep slack

# 4. skip reverse-DNS (faster, IPs only — useful on a slow resolver)
sudo bandwhich -n

# 5. show only the totals pane and exit after 60 seconds (CI / scripted)
timeout 60 sudo bandwhich --raw > /tmp/bw.log

# 6. answer "who just spiked my upload?" in 5 seconds
# launch bandwhich, watch the Processes pane, sort by upload column
```

## Niche It Fills

**Per-process, per-host bandwidth attribution at the terminal.**
Most network tools fall on one side of a fence: `iftop` /
`nload` / `vnstat` show interface-level totals (you know the pipe
is full, not who is filling it); `lsof -i` / `ss -tulpan` /
`netstat` show *connections* but not *bytes per second*. The
overlap zone — "rank live processes by how much bandwidth they
are spending right now, and to which remote hosts" — is where
`bandwhich` lives. `nethogs` is the closest peer; `bandwhich`
adds the remote-host pane and a friendlier TUI.

## Why use it

Three things `bandwhich` does that the classic tools do not,
that pay back the install:

1. **Process + host + connection in one screen.** The three
   panes update in lockstep so you immediately correlate "this
   process" → "this remote" → "this 5-tuple," instead of
   `nethogs` (process only) followed by `ss -tnp` (connection
   only) followed by `dig -x` (host only).
2. **Reverse-DNS by default.** Hostnames not just IPs — you see
   `edgeapi.slack.com` not `54.x.y.z`. The lookup is async with a
   small cache so it does not stall the TUI; opt out with `-n`.
3. **`--raw` is `bandwhich` for scripts.** Streams a stable
   line-per-connection format suitable for `grep` / `awk` / Loki
   ingestion, so the same binary that powers the TUI also feeds
   ad-hoc "what did egress look like during the incident
   window?" log analysis.

## Vs Already Cataloged

- **Vs [`bottom`](../bottom/):** overlap but different focus —
  `bottom` is a general system monitor (CPU, mem, disk, net,
  GPU, processes, batteries) with an *interface-level* network
  view. `bandwhich` is purpose-built for *per-process and
  per-remote-host* network attribution. Use `bottom` to see "the
  box is healthy except the network is hot," then `bandwhich` to
  see *which* process / *which* remote.
- **Vs [`procs`](../procs/) / [`gping`](../gping/):** orthogonal
  — `procs` is a process-table viewer (no live network stats),
  `gping` is a multi-host latency grapher (no per-process
  attribution). `bandwhich` answers a different question.
- **Vs `nethogs` (not cataloged):** the closest peer — both rank
  processes by bandwidth. `nethogs` is older, smaller, and
  Linux-only; `bandwhich` adds the remote-host pane, the
  connections pane, reverse-DNS, the `--raw` machine-readable
  mode, and macOS / Windows support. Keep `nethogs` if it is
  already in your sysadmin muscle memory; pick `bandwhich` for
  cross-platform and for the host-level breakdown.
- **Vs `iftop` / `nload` (not cataloged):** orthogonal —
  interface-level totals only. They tell you the pipe is full;
  `bandwhich` tells you who filled it.

## Caveats

- **Requires elevated privileges.** Raw packet capture needs
  `cap_net_raw` on Linux or root via `sudo` on macOS. Granting
  the capability once is the cleanest path; running everyday
  monitoring under `sudo` is a footgun (terminal escape →
  privilege).
- **PID resolution is best-effort on macOS.** Sandboxed apps and
  short-lived connections may show as "<UNKNOWN>" because the
  socket → PID mapping closed before `bandwhich` could read it.
  Linux is more reliable thanks to `/proc/net/tcp` + `/proc/<pid>/fd`.
- **Reverse-DNS depends on your resolver.** A slow or
  rate-limited DNS path will show IPs for a while before names
  appear; pass `-n` to skip resolution entirely on flaky
  networks.
- **No historical store.** `bandwhich` is live-only — it does
  not keep a rolling history beyond the running session. For
  retrospective bandwidth accounting use `vnstat` (interface
  totals) or pipe `--raw` to a log sink.
- **VPN / WireGuard interfaces need explicit `-i`.** The
  auto-detected interface is usually the physical NIC; if your
  traffic is tunnelled, point at `utun*` (macOS) or `wg0`
  (Linux) explicitly.
- **Windows needs npcap.** Install npcap with WinPcap-compat
  mode before running, otherwise the capture handle will fail
  to open.
