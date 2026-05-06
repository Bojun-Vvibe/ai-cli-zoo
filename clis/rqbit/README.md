# rqbit

> **Single-binary Rust BitTorrent client + daemon + Web UI** —
> downloads (and seeds) torrents from a `.torrent` file, a
> magnet link, or an info-hash, exposes the same surface three
> ways (one-shot CLI, long-lived HTTP+JSON daemon on `:3030`,
> bundled WebUI served from the daemon), and ships as one
> ~10 MB static binary with zero runtime dependencies. Pinned
> to **v9.0.0-beta.2** (released 2026-01-20,
> [`gh api repos/ikatson/rqbit/releases/latest`](https://github.com/ikatson/rqbit/releases/latest),
> [LICENSE](https://github.com/ikatson/rqbit/blob/main/LICENSE),
> Apache-2.0).

Source: <https://github.com/ikatson/rqbit>

## TL;DR

The historical "headless BitTorrent on a Linux box" answer is
some combination of `transmission-daemon` (C, mature, but a
multi-process install with separate `transmission-cli` /
`transmission-remote` / `transmission-web` packages and a config
file shaped by 2008), `qbittorrent-nox` (C++/Qt, heavy install
even for the no-X11 build, daemonised qBittorrent main loop), or
`aria2c` with `--bt-*` flags (great general downloader, but its
BitTorrent layer is the secondary use case). `rqbit` is the
modern Rust take on the same niche: one static binary covers the
client (`rqbit download magnet:...`), the long-lived daemon
(`rqbit server start /downloads/`), and the WebUI (the daemon
serves a React UI from the same port that exposes its JSON API),
so a Raspberry Pi / NAS / seedbox install is one `scp` of one
binary plus a systemd unit. Speaks the standard BitTorrent v1
protocol (BEP-3) plus the table stakes — DHT (BEP-5) for
trackerless swarms, uTP (BEP-29) over UDP for ISP-friendly
congestion control, PEX, magnet links, IPv6, and uPnP/NAT-PMP
for port forwarding — and the JSON HTTP API (`POST /torrents`,
`GET /torrents/{id}/stats/v1`) is the integration surface for
custom dashboards / Home Assistant / cron-shaped automation
without parsing the WebUI's HTML.

## Install

```bash
# Cargo (any platform with a Rust toolchain)
cargo install rqbit --locked

# Pre-built binary from a release (Linux / macOS / Windows)
curl -L \
  https://github.com/ikatson/rqbit/releases/download/v9.0.0-beta.2/rqbit-linux-static-x86_64 \
  -o /usr/local/bin/rqbit && chmod +x /usr/local/bin/rqbit

# Docker
docker run --rm -p 3030:3030 -v /downloads:/downloads \
  ikatson/rqbit server start /downloads

# verify
rqbit --version
```

## Representative examples

```bash
# 1. One-shot download a magnet link to ./out
rqbit download -o ./out 'magnet:?xt=urn:btih:...'

# 2. Long-lived daemon + WebUI on :3030 with /downloads as root
rqbit server start /downloads

# 3. Add a torrent to the running daemon via JSON API
curl -X POST http://localhost:3030/torrents \
  --data 'magnet:?xt=urn:btih:...'

# 4. Watch live transfer stats from the daemon
curl -s http://localhost:3030/torrents/0/stats/v1 | jq

# 5. List active torrents
curl -s http://localhost:3030/torrents | jq '.torrents[] | {id,name,state}'

# 6. Headless seed-only mode (no fetch, just announce + serve pieces)
rqbit server start --disable-dht-persistence /seed/
```

## When to use vs. alternatives

- Pick **rqbit** when the deployment is a Linux/macOS/Windows
  host where "one static binary + one systemd unit + one open
  port" beats a packaged daemon with config-file gymnastics, and
  the WebUI / JSON API together cover both the human and the
  automation surface.
- Pick **transmission-daemon** when the host already runs the
  Transmission ecosystem (`transmission-remote` clients on
  laptops, existing watch directories, RPC-based wrappers like
  Sonarr/Radarr that speak the Transmission RPC dialect) — those
  integrations are not portable to rqbit's JSON API.
- Pick **qbittorrent-nox** when the operator wants the qBittorrent
  WebUI specifically (long-established, heavy feature set:
  RSS, search plugins, scheduler, IP filter, categories), and
  the C++/Qt install footprint is acceptable.
- Pick [`aria2`](../aria2/) when BitTorrent is one of *several*
  protocols the same downloader must speak (HTTP, HTTPS, FTP,
  SFTP, Metalink, BitTorrent in one daemon) — aria2 wins the
  multi-protocol lane; rqbit wins the BitTorrent-only lane.
- Pick a hosted seedbox / Real-Debrid when the legal / network
  position of the host machine makes running any BitTorrent
  client locally the wrong call — no CLI replaces that policy
  decision.
- Caveats: pre-1.0 (v9 line still tagged `-beta`), so pin the
  binary in CI rather than tracking `@latest`; WebUI auth is
  basic (front it with a reverse proxy + real auth before
  exposing to the public internet); no built-in VPN binding —
  pair with `wg-quick` / `gluetun` / a network-namespace wrapper
  if traffic must egress only via VPN.
