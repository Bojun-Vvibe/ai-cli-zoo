# carbonyl

> Snapshot date: 2026-05. Upstream: <https://github.com/fathyb/carbonyl>

**A full Chromium browser running entirely inside the terminal.**
Carbonyl is a Rust + C++ fork of Chromium that replaces the Skia GPU
backend with a TTY renderer: it rasterizes web pages into half-block
unicode cells (`▀` / `▄`) at the resolution of your terminal, while
keeping the real Blink layout/JS engine intact. The result is the
only "terminal browser" in the catalog that actually runs modern web
apps — WebGL, video (with audio over PulseAudio / CoreAudio), and
SPA frameworks all work — at roughly 60 fps over SSH on a
half-decent VT.

## Repo + version + license

- Repo: <https://github.com/fathyb/carbonyl>
- Latest release: **`v0.0.3`** (2023-02-18)
- License: **BSD-3-Clause** —
  <https://github.com/fathyb/carbonyl/blob/main/license.md>
- License path in repo: `license.md`
- Default branch: `main`
- Language: Rust + C++ (Chromium fork)

## Install

```bash
# Docker (the one-line path; works on macOS / Linux / WSL)
docker run --rm -ti fathyb/carbonyl https://github.com/fathyb/carbonyl

# Open a specific URL with mouse support
docker run --rm -ti fathyb/carbonyl https://news.ycombinator.com

# Run as the system browser (set BROWSER=carbonyl) for tools that
# call `xdg-open` / macOS `open` to surface a URL
export BROWSER="docker run --rm -ti fathyb/carbonyl"
```

## Niche

The "**real Chromium, in your terminal, that actually renders modern
JS-heavy sites**" slot. Where [`browsh`](../browsh/) drives a
headless Firefox over Marionette to produce a low-frame-rate text
preview, where [`bombadillo`](../bombadillo/) and
[`amfora`](../amfora/) are Gemini/Gopher-only by design, where
[`elinks`](../elinks/) and [`w3m`](https://w3m.sourceforge.net/) are
text-mode HTML browsers with no JS engine, Carbonyl ships the *whole*
Blink + V8 + media stack and just swaps the pixel sink. A page that
needs `fetch()` + React + Web Audio works; a page that draws to a
`<canvas>` shows up as half-block pixels in your `tmux` pane. Useful
when:

- You SSH into a box and need to log into an OAuth-only web flow
  without forwarding a graphical session.
- You want to watch a YouTube video over `mosh` for the meme value
  (it actually works — audio routes to the Docker host's audio
  device).
- You're on a remote dev container and the only "browser" available
  is whatever `xdg-open` can find.

## Why it matters

- **Full Blink + V8 + WebGL + WebAudio** — not a curses HTML parser;
  this is Chromium with a terminal raster path. Modern SPAs render
  correctly.
- **Half-block unicode rendering at 60 fps** — uses `▀`/`▄` with
  separate fg/bg colors so each cell encodes two pixels, doubling
  effective vertical resolution; the raster loop is in Rust against
  Chromium's GPU command stream.
- **Mouse support inside the TTY** — clicks, scroll, hover all map
  through `xterm`-style mouse escape sequences.
- **Project status caveat** — last release is 2023-02; the upstream
  Chromium fork is frozen. Treat it as a working novelty / SSH
  rescue tool, not a daily driver. There's no CVE backport pipeline,
  so don't log into your bank in it.
