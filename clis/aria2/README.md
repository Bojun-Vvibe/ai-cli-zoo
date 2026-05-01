# aria2

- **Repo:** https://github.com/aria2/aria2
- **Version:** v1.37.0 (`release-1.37.0`, tagged 2023-11-15 — still the latest stable line as of 2026)
- **License:** GPL-2.0-or-later with OpenSSL exception ([COPYING](https://github.com/aria2/aria2/blob/master/COPYING), with the OpenSSL linking exception spelled out in [`OpenSSL-exception`](https://github.com/aria2/aria2/blob/master/OpenSSL-exception))
- **Language:** C++
- **Install:** `brew install aria2` · `apt install aria2` · `pacman -S aria2` · `dnf install aria2` · prebuilt Windows / Android binaries on the GitHub release page · binary name is `aria2c`

## What it does

`aria2c` is a single-binary, multi-protocol, multi-source download
utility that speaks HTTP/1.1, HTTPS, FTP, SFTP, BitTorrent (BEP-3
plus DHT, PEX, magnet links, Web-Seeding, hybrid v1/v2 torrents)
and Metalink in one process. The defining capability is
**segmented downloads from multiple URIs in parallel**: give it
five mirrors of the same ISO and it splits the byte range across
them, automatically retrying failed segments against the
surviving mirrors. The same engine drives torrent swarms, FTP
mirrors, and plain HTTP — so a single `aria2c` invocation can
mix HTTP and torrent sources for the same file and pick whichever
arrives first per chunk. It exposes a JSON-RPC and XML-RPC
control surface on a local port (`--enable-rpc`), which is what
WebUI front-ends like AriaNg and the various mobile remotes
talk to; the same RPC surface makes it easy to drive from a
script without parsing CLI output.

## When to pick it / when not to

Reach for `aria2c` when you need to **pull a large artifact
quickly over a flaky link**, when you have multiple mirrors and
want segment-level resilience without writing retry logic, or
when a job needs to run as a long-lived daemon controlled over
RPC (queue files, pause/resume, rate-limit on the fly). It's
the right answer for CI jobs that fetch big model weights from
several CDN mirrors, for desktop "download manager" replacements,
and for mixed HTTP+torrent distribution flows where the torrent
swarm is your fallback CDN.

Skip it when a single `curl -O` would do — the `aria2c` config
surface and segmented-download bookkeeping are overkill for a
1 MB JSON fetch. Skip it for HTTP-API scripting (where `curl`
or [`xh`](../xh/) read better in pipelines) and for
browser-style sessions with cookies, redirects, and JavaScript
(use a headless browser). Note: the v1.37.0 cadence is slow —
the project is in maintenance, not active feature development —
which is fine for "downloads files reliably" but means do not
expect HTTP/3 or QUIC support soon.

## Why it matters in an AI-native workflow

Model fetches are the bottleneck of every local-inference
pipeline: a 70B-parameter weight set is 40–140 GB across
many shards, often pulled from two or three mirrors that each
throttle individual connections. `aria2c -x 16 -s 16 -j 4`
saturates the link by parallelizing across both segments and
files, and `--checksum` validates each shard against the
manifest as it lands. The RPC interface lets an agent enqueue
and monitor downloads as a side effect rather than blocking
on shell output, which fits the "long-running fetch as a
managed background task" pattern that agent loops increasingly
need.

## Example invocations

```bash
# Resume-capable parallel HTTP download with 16 segments
aria2c -x 16 -s 16 -c https://example.com/big.iso

# Multiple mirrors of the same file — segment across them
aria2c -x 16 \
  https://mirror1.example/file.tar.zst \
  https://mirror2.example/file.tar.zst \
  https://mirror3.example/file.tar.zst

# Torrent (magnet link), with seeding disabled after completion
aria2c --seed-time=0 'magnet:?xt=urn:btih:...'

# Batch download from a list, four files in parallel
aria2c -j 4 -i urls.txt

# Run as RPC daemon, control from a WebUI or another process
aria2c --enable-rpc --rpc-listen-all=false --rpc-listen-port=6800 \
  --rpc-secret=$(openssl rand -hex 16) --daemon

# Verify against a checksum as it downloads
aria2c --checksum=sha-256=abcdef... https://example.com/file.bin
```

## Alternatives in this catalog

- [`yt-dlp`](../yt-dlp/) — purpose-built media downloader; reach
  for it when the source is a streaming site, not a plain URL.
- [`monolith`](../monolith/) — saves a single web page as one
  self-contained HTML; different problem (page archive, not
  bulk file fetch).
- [`xh`](../xh/) — modern HTTP client for ad-hoc requests;
  use it for API calls, use `aria2c` when bytes-per-second is
  the constraint.
