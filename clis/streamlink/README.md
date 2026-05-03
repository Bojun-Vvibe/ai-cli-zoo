# streamlink

> **A CLI that takes a *web page URL* of a live
> video stream and hands the underlying media
> stream to a player of your choice** — `streamlink
> https://twitch.tv/somechannel best` pipes
> the live HLS to `mpv`, no browser, no Flash, no
> proprietary client. ~150 site-specific plugins
> ship in-tree (Twitch, YouTube live, Kick, every
> major broadcaster's IPTV portal, BBC iPlayer,
> the Olympics-class network feeds, etc.).
> Pinned to **8.3.0**
> ([LICENSE](https://github.com/streamlink/streamlink/blob/master/LICENSE),
> BSD-2-Clause).

Source: <https://github.com/streamlink/streamlink>

## TL;DR

Browsers are bad at watching long live streams.
The page eats RAM, the player has ads, the player
has DRM-shaped UI, the player buffers wrong, the
page burns a CPU core on decoding into a tiny
`<video>` element you then fullscreen anyway.
`streamlink` skips the page entirely. It knows,
per-site, how to find the actual HLS / DASH /
RTMP manifest, fetches it, and pipes the chosen
quality (`best`, `worst`, `720p`, `audio_only`)
to the player binary you already use — `mpv`,
`vlc`, `ffplay`, or `--stdout` for further
piping into `ffmpeg` for recording. It is the
"`yt-dlp` for *live*" — and where they overlap
on VOD, `streamlink`'s focus is real-time
playback, `yt-dlp`'s is download.

## Install

```bash
# pipx (recommended — isolated venv)
pipx install streamlink

# pip (user)
python3 -m pip install --user streamlink

# Homebrew
brew install streamlink

# Verify
streamlink --version    # streamlink 8.3.0
```

Pure-Python; the only hard runtime is Python ≥
3.9. A media player (`mpv` strongly
recommended; `vlc` / `ffplay` also fine) is the
peer dependency on the *playback* side; nothing
is bundled.

## Use it for

```bash
# 1) Watch a live stream in mpv at the best quality
streamlink https://twitch.tv/somechannel best
#   → mpv opens, starts playing.
#     `best` resolves to the highest variant the
#     site advertises (1080p60, 4k, etc).

# 2) Pick a specific quality
streamlink https://twitch.tv/somechannel 720p
streamlink https://twitch.tv/somechannel audio_only
streamlink https://twitch.tv/somechannel worst

# 3) List qualities without playing
streamlink https://twitch.tv/somechannel
#   → Available streams: audio_only, 160p, 360p,
#     480p, 720p, 720p60, 1080p60 (worst, best)

# 4) Record to disk while streaming
streamlink -o stream.ts \
  https://twitch.tv/somechannel best
# Press Ctrl-C to stop. Output is raw transport
# stream; remux with `ffmpeg -i stream.ts -c copy
# stream.mp4` if you want MP4.

# 5) Pipe to ffmpeg (transcode / record / restream)
streamlink --stdout https://twitch.tv/somechannel best \
  | ffmpeg -i - -c:v libx264 -preset veryfast \
      -c:a aac -f flv rtmp://relay.example/live/key

# 6) Pick a player explicitly
streamlink -p vlc https://twitch.tv/somechannel best
streamlink -p 'mpv --cache=yes --hwdec=auto' \
  https://twitch.tv/somechannel best

# 7) Authenticate where the site requires it
#    (each plugin documents its auth flags)
streamlink --twitch-api-header 'Authorization=OAuth XXX' \
  https://twitch.tv/somechannel best

# 8) Twitch-specific: skip pre-roll ads, request
#    low-latency playback
streamlink --twitch-disable-ads --twitch-low-latency \
  https://twitch.tv/somechannel best

# 9) Persistent config — keep flags out of muscle memory
mkdir -p ~/.config/streamlink
cat > ~/.config/streamlink/config <<'EOF'
default-stream=best
player=mpv --cache=yes --hwdec=auto
twitch-disable-ads
twitch-low-latency
EOF
streamlink https://twitch.tv/somechannel    # uses defaults

# 10) Per-site overrides (config.<plugin>)
cat > ~/.config/streamlink/config.twitch <<'EOF'
twitch-api-header=Authorization=OAuth XXX
EOF
```

## Why include it in a CLI catalog

1. **It is the canonical "browserless live
   stream" tool.** No other widely-deployed CLI
   covers the live / IPTV / broadcaster space
   the same way. Where `yt-dlp` excels at *video
   downloads*, `streamlink` excels at *live
   playback piped into your player of choice*.
2. **The plugin model is right-sized.** ~150
   site plugins live in-tree, get patched
   together with each release, and degrade
   gracefully (when a site changes its
   manifest format, one plugin breaks; the rest
   keep working). Adding a private plugin is a
   single-file Python module dropped into
   `~/.local/share/streamlink/plugins/`.
3. **The "stream-as-pipe" model composes
   beautifully.** `--stdout` lets `streamlink`
   become the front-end of any media pipeline:
   record with `ffmpeg`, restream with `nginx-
   rtmp`, transcode for low-bandwidth viewers,
   sample frames for an ML classifier, run a
   vision model over a 24/7 webcam feed. The
   CLI is the manifest-resolver; everything
   downstream is your call.

For an LLM-CLI workflow, `streamlink` is the
"point a model at a live video signal" piece:
`streamlink --stdout <url> 480p | ffmpeg -i -
-vf fps=1 -f image2 frame_%d.jpg` gives you a
1-FPS image stream from any supported live
source, ready to feed into a vision model for
event detection, transcript bootstrapping (paired
with `whisper`), or operational monitoring of a
public webcam / livestream. The "live" half of
the AV pipeline that `yt-dlp` doesn't cover.

## Vs Already Cataloged

- **Vs [`yt-dlp`](../yt-dlp/):** sibling but
  different focus. `yt-dlp` is a *downloader*
  — its job is to land the entire video as a
  file on disk, with subtitle extraction,
  thumbnail embed, format selection, post-
  processing. `streamlink` is a *live player
  front-end* — its job is to pipe a continuous
  manifest into a player or `ffmpeg` while it
  is still being broadcast. The overlap is
  "VOD URLs" where both work; the disjoint
  parts are "live broadcasts of arbitrary
  duration" (streamlink only) and "post-
  processed file with chapters and metadata"
  (yt-dlp only). Real workflows often use
  both: streamlink for monitoring, yt-dlp for
  archival.
- **Vs [`gallery-dl`](../gallery-dl/):**
  orthogonal. `gallery-dl` is for *images*
  (booru sites, social-media galleries,
  art sites). `streamlink` is for *live
  video*. They occupy different corners of
  the "scrape media off a website" space.
- **Vs `ffmpeg` directly:** `ffmpeg` *can*
  read an HLS / DASH manifest URL — but only
  if you already have the manifest URL.
  `streamlink`'s value is the per-site logic
  that *finds* the manifest URL behind a
  page, handles auth, signs URLs where the
  site requires it, picks the right CDN, and
  routes through the right ad-bypass flow.
  Once you have the manifest, `ffmpeg`
  alone would work — but you don't, and
  reverse-engineering it per site is exactly
  what `streamlink`'s plugins do for you.
- **Vs browser-based capture** (browser
  extension that records the `<video>`
  element): completely different mechanism.
  `streamlink` doesn't need a browser, doesn't
  need JS execution for most plugins, doesn't
  pay the GPU cost of rendering the page.

## Caveats

- **Plugin breakage is a fact of life.** When
  a site changes its manifest format, the
  matching plugin breaks until upstream patches
  it. The fix cycle is usually fast (days, not
  months) but you should be on a recent release
  — pin to a major and `pipx upgrade
  streamlink` regularly rather than freezing on
  a stale version.
- **Streamlink does not bypass DRM.** Sites
  using Widevine / FairPlay / PlayReady cannot
  be played by `streamlink` (or by `ffmpeg`,
  or by any other CLI). The supported sites are
  the ones that ship plain HLS / DASH.
- **Respect the source's terms of service.**
  `streamlink` is a player; it doesn't ask the
  site for permission. Some plugins document
  rate-limit / ToS notes in `streamlink
  --plugins` output — read them before
  long-running automation.
- **`--twitch-disable-ads` works by skipping
  ad segments at the manifest level.** It's
  technically reliable, ethically a choice;
  if you would tip the streamer for an
  ad-supported view, do that separately.
- **The Python startup cost is not zero.**
  Cold-start to first frame is typically 1–3
  seconds (manifest fetch + player launch +
  HLS prebuffer). For a long live session
  this is invisible; for "spawn a stream
  every second" automation, batch differently.
- **BSD-2-Clause license**
  ([LICENSE](https://github.com/streamlink/streamlink/blob/master/LICENSE))
  — permissive; safe to ship inside a
  proprietary product with attribution.
