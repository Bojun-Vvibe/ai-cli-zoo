# menyoki

> **Screen capture / screencast / GIF recorder driven entirely
> from the terminal** — a single Rust binary that records a
> region, window, or full screen on X11 and writes the result
> as an animated `.gif`, `.apng`, `.webp`, `.mp4`, `.gifski`-
> optimised GIF, or a still `.png` / `.jpg` / `.bmp` / `.tiff`
> / `.farbfeld` / `.pnm`, with frame rate, duration, key
> bindings, mouse-cursor inclusion, and post-processing
> (resize, crop, rotate, flip, blur, padding, palette
> reduction) all controlled by flags — no GUI, no daemon, no
> click-to-record. Pinned to **v1.7.0** (commit
> `f8b4ebe9ef82a193f929b9869eb903fa8eb8af61`,
> [LICENSE](https://github.com/orhun/menyoki/blob/master/LICENSE),
> SPDX: `GPL-3.0`).

Source: <https://github.com/orhun/menyoki>

## TL;DR

`menyoki` (Turkish for "monkey") records the screen and writes
animated images / video without ever touching a GUI: `menyoki
record --fps 20 --duration 8 gif --output demo.gif` selects a
window with the mouse, records 8 seconds at 20 fps, encodes
to GIF, runs `gifski`-style palette optimisation, and writes
`demo.gif` — all from one shell invocation. The capture target
is X11 (`X11` / `Xorg` / Xwayland), the encoder pipeline is
pure-Rust (`gif`, `png`, `apng`, `image`, `webp`), and every
knob (frame rate, duration, key-stop binding, alpha, region,
padding, post-effects) is a CLI flag — so the tool composes
into Makefiles, `just` recipes, CI smoke-tests of TUIs, and
README-asset pipelines without manual clicks.

## Install

```bash
# Cargo (any OS with X11)
cargo install menyoki

# Arch
pacman -S menyoki

# Pre-built binary (Linux x86_64 / aarch64)
# https://github.com/orhun/menyoki/releases/tag/v1.7.0

# verify
menyoki --version    # menyoki 1.7.0
```

`menyoki` requires X11 libraries at runtime (`libxrandr`,
`libxext`); on Wayland sessions run via Xwayland.

## License

GPL-3.0 — see
[LICENSE](https://github.com/orhun/menyoki/blob/master/LICENSE).
Copyleft: any redistribution of modified `menyoki` binaries
must ship source under the same licence; using the binary as
an unmodified tool in a recording pipeline imposes no
obligations on the recorded artefacts.

## Representative Commands

```bash
# 1. record an interactively-selected window to GIF, 8 s at 20 fps
menyoki record --fps 20 --duration 8 gif --output demo.gif

# 2. capture a fixed region (x,y,w,h) and write a still PNG
menyoki capture --size 1280x720 --pos 100,100 png --output shot.png

# 3. record full screen, stop on Ctrl+D, encode to APNG
menyoki record --root --action-keys ctrl-d apng --output session.apng

# 4. record + post-process (resize to 600px wide, 6-colour palette)
menyoki record --duration 5 --resize 600x0 --quality 6 \
  gif --output small.gif

# 5. screenshot every 30s into timestamped files (cron / loop)
while sleep 30; do
  menyoki capture --root png --output "shot-$(date +%s).png"
done

# 6. record without including the mouse cursor
menyoki record --no-cursor --duration 4 gif --output clean.gif
```

## Why It Matters

Recording a terminal demo, a TUI bug repro, or a small UI
interaction for a README / issue / Slack thread is a recurring
chore, and the usual answers are heavy: macOS Screenshot →
QuickTime → ffmpeg conversion, OBS Studio with a scene set
up, or a desktop tool like `peek` that wants a windowed GUI.
`menyoki` collapses the loop into one CLI: pick a window,
record N seconds at M fps, get an optimised GIF / APNG / WebP
straight to disk. Because every knob is a flag and there is
no GUI handshake, it slots into automation — `just demo-gif`
recipe that re-records the README banner, CI job that captures
a TUI smoke-test as an artefact, post-commit hook that updates
animated docs. Pairs with [`asciinema`](../asciinema/) (text
recording, much smaller files, terminal-only) and
[`vhs`](../vhs/) / [`asciinema-agg`](../asciinema-agg/)
(scripted terminal GIFs from a `.tape` file). Reach for
`menyoki` when the recording target is an *arbitrary X11
window or screen region* — a TUI plus a popup, a
window-manager interaction, a video frame, anything outside a
single terminal — and you want the full encoder pipeline
(format choice, palette, resize, crop, alpha, padding, frame
rate) selectable per-invocation. The killer property is
**fully headless screen capture**: no GUI to launch, no daemon
to keep alive, just `menyoki record … --output foo.gif` and a
file appears, identical between a developer laptop and a
remote X11 session over `ssh -X`.
