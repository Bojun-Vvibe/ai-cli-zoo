# mediamtx

> **Single-binary real-time media server that ingests + serves the
> same live stream over RTSP, RTMP, HLS, WebRTC, and SRT
> simultaneously** — drop one ~26 MB Go binary on a host, point it
> at a `mediamtx.yml`, and an OBS push (RTMP), an IP camera (RTSP),
> a `ffmpeg` SRT contribution feed, or a browser WebRTC publisher
> can all hit the *same* path and any consumer can pull it back in
> any of those five protocols (plus HLS-LL for sub-second latency to
> a `<video>` tag); no Wowza license, no Janus build, no nginx-rtmp
> patching — pinned to **v1.18.1** (released 2026-04-30,
> [LICENSE](https://github.com/bluenviron/mediamtx/blob/v1.18.1/LICENSE),
> SPDX `MIT`).

Source: <https://github.com/bluenviron/mediamtx>

## TL;DR

Real-time video plumbing has historically been one tool per
protocol: **nginx-rtmp** for RTMP ingest, **GStreamer
`rtsp-server`** for RTSP, **Janus** / **mediasoup** for WebRTC,
**Wowza** / **AWS MediaLive** for the commercial all-in-one,
**HLS.js + a CDN** for the playback side. Stitching those
together to do "OBS pushes RTMP → camera pulls back RTSP →
browser pulls back WebRTC, all from the same source, with
sub-second latency on the browser side" is several days of glue
config and at least three daemons.

`mediamtx` (born as `rtsp-simple-server`, renamed in 2023 once
the protocol surface outgrew RTSP) collapses the lot into one
process. Each named `path` in `mediamtx.yml` is a logical stream;
the server demuxes the codec packets once on ingest and remuxes
on the fly per consumer protocol — H.264 / H.265 / AV1 / VP8 /
VP9 video and Opus / AAC / G711 / G722 / MPEG-4 Audio / LPCM
audio plus KLV metadata, with the codec matrix per protocol
documented in the README. The HTTP API exposes `paths`,
`recordings`, `hooks` (`runOnReady`, `runOnDemand`,
`runOnRecord`) for "ffmpeg-pull-this-RTSP-camera-on-first-viewer"
patterns, and the on-disk recording format is fragmented MP4 /
fMP4 ready to serve from any static-file CDN.

## Install

```bash
# macOS (Homebrew)
brew install mediamtx

# Pre-built tarballs from upstream tags (Linux amd64 / arm64 /
# armv6 / armv7, macOS amd64 / arm64, Windows amd64) live at:
#   https://github.com/bluenviron/mediamtx/releases/tag/v1.18.1

# Docker (official multi-arch image)
docker run --rm -it --network=host \
  -v ./mediamtx.yml:/mediamtx.yml \
  bluenviron/mediamtx:1.18.1

# Plain binary install on a Linux host (no systemd unit shipped —
# wrap with your supervisor of choice)
curl -L https://github.com/bluenviron/mediamtx/releases/download/v1.18.1/mediamtx_v1.18.1_linux_amd64.tar.gz \
  | tar xz && sudo install -m 0755 mediamtx /usr/local/bin/
```

Default config (zero-byte `mediamtx.yml`) opens RTSP on `:8554`,
RTMP on `:1935`, HLS on `:8888`, WebRTC HTTP on `:8889`, SRT on
`:8890`, and the metrics endpoint on `:9998`. Every protocol can
be disabled per stanza in the YAML.

## Common invocations

```bash
# Smoke-test: run with default config, accept any push to any path
mediamtx

# Push from OBS (Service: Custom, Server: rtmp://host/live, Key: cam1)
# then pull as HLS in a browser:
#   http://host:8888/cam1/index.m3u8

# Pull an existing IP camera on demand (only when first viewer arrives)
# In mediamtx.yml:
#   paths:
#     cam1:
#       source: rtsp://192.168.1.20:554/Streaming/Channels/101
#       sourceOnDemand: yes

# Record every stream that lands on `paths.all_others` to fMP4 segments
#   pathDefaults:
#     record: yes
#     recordPath: /var/lib/mediamtx/recordings/%path/%Y-%m-%d_%H-%M-%S
#     recordFormat: fmp4
#     recordSegmentDuration: 1h

# Browser WebRTC viewer (zero plugins, sub-second latency)
#   http://host:8889/cam1

# Re-publish ffmpeg test pattern as RTSP
ffmpeg -re -f lavfi -i testsrc -c:v libx264 -f rtsp \
  rtsp://localhost:8554/test
```

## Why orthogonal to existing zoo

The zoo has **zero** real-time-media servers. The closest
adjacent tools are `streamlink` (a *client* that scrapes streaming
sites for the underlying HLS / DASH and pipes it to a local
player — opposite direction), `sox` (audio file processing — no
network protocol), `vhs` / `asciinema` (terminal recording, not
A/V), and the various WebSocket / HTTP probes (`websocat`, `xh`)
which can poke a media endpoint but cannot demux a codec stream.
For anyone setting up an RTSP camera grid, an OBS-to-browser
streaming pipeline, an SRT contribution feed for live events, or
a WebRTC playback layer for low-latency monitoring, this is a
greenfield niche in the catalog.

## Caveats

- The all-protocols-from-one-binary pitch is real, but each
  protocol has its own codec matrix — RTMP cannot carry H.265 or
  AV1 in the standard FLV container (Enhanced RTMP / E-RTMP works
  with newer OBS but not all clients); HLS-LL needs the
  `lowLatencyMode: yes` switch *and* a player that speaks the
  HLS-LL protocol; WebRTC is H.264 / VP8 / VP9 / AV1 + Opus only
  (no AAC). Check the README's "Codec table" before assuming a
  codec is end-to-end.
- The HTTP API + hooks (`runOnReady`, `runOnDemand`,
  `runOnRecord`) execute arbitrary shell commands — historically
  there have been query-string injection CVEs (the v1.18.1
  changelog itself fixes one with `MTX_QUERY` URL-encoding); keep
  the API bound to localhost behind a reverse proxy and audit
  hook scripts as you would any sudoers entry.
- Recording lands as fragmented MP4 / fMP4 segments (or
  MPEG-TS); for a "single mp4 of the whole event" you concat /
  remux post-hoc with `ffmpeg`. There is no built-in DVR /
  rewind UI.
- Authentication ships as basic-auth + JWT + per-path
  user/password matrices in YAML; for SSO, group policies, or
  signed-URL CDN-style auth you wire it in front via nginx /
  Cloudflare / etc. There is no OIDC integration in-tree.
- It is a *server* and wants exclusive ownership of its TCP / UDP
  ports — running alongside a separate nginx-rtmp on `:1935`,
  Janus on the same WebRTC ports, or another RTSP daemon on
  `:8554` requires moving one of them. The v1 ports are stable
  and documented.
- Single-node only — there is no built-in clustering / load
  balancing. For large fan-outs, put it behind a CDN (the HLS
  endpoint is a static-file pattern; WebRTC needs a proper SFU
  in front for high-viewer-count scenarios).
